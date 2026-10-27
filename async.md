<!-- Draft reply to lrbison's comment on NVIDIA/nccl issue #2509 —
     paste as javioustlj on GitHub. Do not post automatically. -->

Hi @lrbison, thanks for taking a look!

Your "poll the new communicator" model is exactly how `ncclCommGrow` works —
its job is launched on the new comm
([src/init.cc#L3840-L3846](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3840-L3846))
and `*newcomm` is assigned synchronously before return
([src/init.cc#L3808](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3808)),
so the caller holds a valid handle to poll. But split/shrink differ in two
ways that matter for this issue:

### 1. For split/shrink the pending state lives on the parent, not the child

The split/shrink job is launched on the **parent**
([src/init.cc#L3627-L3633](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3627-L3633) —
the base `job->comm` is the parent). At non-blocking group end it is therefore
the parent that is stamped `ncclInProgress`
([src/group.cc#L1107](https://github.com/NVIDIA/nccl/blob/master/src/group.cc#L1107),
via `job->comm`), and the background thread later resets it to the final result
([src/group.cc#L876-L883](https://github.com/NVIDIA/nccl/blob/master/src/group.cc#L876-L883)).
The child's `asyncResult` is never set to `ncclInProgress` by the split path —
`ncclCommGetAsyncError` reads `comm->asyncResult`
([src/init.cc#L3948](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3948)),
and that field's only writer in the whole tree is `ncclCommSetAsyncError`
([src/init.cc#L3043](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3043)),
none of whose call sites passes the split child. The child is calloc'd
(`ncclSuccess == 0`), so polling it reports `ncclSuccess` from the moment it
exists. (The child does track its own init state — in `initState`, set to
`ncclInProgress` at
[src/init.cc#L3606](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3606)
and updated by the background thread — but `ncclCommGetAsyncError` never reads
`initState`, so that state is not observable through the polling API. The
reason the parent is the stamped one is the async-job field layout: the derived
`ncclCommInitRankAsyncJob::comm` (the child, set at :3610) shadows the base
`ncclAsyncJob::comm`, which is the field `ncclAsyncLaunch` sets from the comm
the job is *launched on* — the parent. Group-end stamping and the background
thread's final result both target that base field, i.e. the parent.)

The user guide's own non-blocking split example polls the parent:

```c
CHECK(ncclCommSplit(comm, color, key, &childComm, &config));
do {
  CHECK(ncclCommGetAsyncError(comm, &state));
} while (state == ncclInProgress && checkTimeout() != true);
```

(docs/userguide/source/usage/communicators.rst, "abort" section)

So after polling the parent, the extra `ncclCommGetAsyncError(w->newcomm, ...)`
loop in your diff returns `ncclSuccess` immediately — it doesn't actually wait
for anything.

### 2. At return time the caller has no new-comm handle to poll

`*newcomm` is set to `NCCL_COMM_NULL` synchronously
([src/init.cc#L3568](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3568))
and is only populated by the background thread when init finishes
([src/init.cc#L2244](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L2244),
through the pointer captured at
[src/init.cc#L3611](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3611)).
"The new communicator has been allocated" is true internally, but the caller's
`newcomm` is still `NCCL_COMM_NULL` at that point — polling it is a NULL-comm
call, which `CommCheck` rejects with `ncclInvalidArgument`. The parent is the
only pollable handle. (For `NCCL_SPLIT_NOCOLOR` no child comm is allocated at
all, [src/init.cc#L3569-L3570](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3569-L3570),
so `w->newcomm` stays `NCCL_COMM_NULL` forever.)

### 3. On NOCOLOR: the race is the write, not the value

The repro deliberately uses NOCOLOR because it's the clearest case of a caller
that keys storage lifetime on the return value ("no communicator was created,
nothing more will happen, release the storage"). But switching to `color=0`
does not remove the race: the background thread still writes through the
captured pointer at
[src/init.cc#L2244](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L2244)
after the API has returned. Any free between the return and that write is a
use-after-free, regardless of whether the value written is NULL (NOCOLOR) or a
comm handle (`color=0`).

### 4. The return value is inconsistent within the same family

`ncclCommInitRankConfig` itself returns `ncclInProgress` in non-blocking mode —
your own diff accepts it (`if (ret != ncclSuccess && ret != ncclInProgress)
return 1;`) — and so do `ncclCommInitRankScalable`, `ncclCommFinalize`,
`ncclCommRevoke`, and the enqueue collectives, all by re-reading the async
state via `ncclCommGetAsyncError` after the group end
([src/init.cc#L3074](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3074),
[:3111](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3111),
[:3245](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3245),
[:3475](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3475),
[src/enqueue/enqueue.cc#L3522](https://github.com/NVIDIA/nccl/blob/master/src/enqueue/enqueue.cc#L3522)).
Split/shrink are the only ones that swallow it, purely because of which check
macro was typed at the group-end call
([src/init.cc#L3671](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3671),
[:3893](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L3893)) — the
difference looks accidental rather than designed.

### 5. The two poll shapes are not equivalent — and only one is safe today

Consider the two natural ways to write the parent poll (in both, `res` first
holds the `ncclCommSplit` return value):

```c
ncclResult_t res = ncclCommSplit(parent, 0, 0, &w->newcomm, NULL);

/* (a) poll-anyway — the shape every user guide example uses */
do {
  CHECK(ncclCommGetAsyncError(parent, &res));
} while (res == ncclInProgress);

/* (b) keyed on the API return value */
while (res == ncclInProgress) {
  CHECK(ncclCommGetAsyncError(parent, &res));
}
```

(a) never consults the return value. The parent is stamped `ncclInProgress`
at group end, before the API returns
([src/group.cc#L1107](https://github.com/NVIDIA/nccl/blob/master/src/group.cc#L1107)),
so the unconditional first poll always observes the pending state — or, if
the background thread has already finished, the final state, and by then
`*newcomm` is already populated: the store through the captured pointer
([src/init.cc#L2244](https://github.com/NVIDIA/nccl/blob/master/src/init.cc#L2244))
happens before the background thread stamps the parent's final result
([src/group.cc#L880](https://github.com/NVIDIA/nccl/blob/master/src/group.cc#L880)).
Shape (a) is therefore correct regardless of what the API returns — today,
without any fix. A caller following the user guide's split example verbatim
does not hit this race. And it is the only shape the user guide ever shows:
every example (`ncclCommInitRankConfig`, split, and grow) polls with a
do-while on a separate state variable, deliberately not depending on the
return value.

(b) tests the return value first. Under the current behavior the API returns
`ncclSuccess` while the background init is still running, so the loop body
never runs and the caller proceeds with `*newcomm` still `NCCL_COMM_NULL` /
being written concurrently — the failure mode from the issue. Shape (b) only
becomes correct once the return value is truthful, i.e. with the fix below.

The repro's storage-lifetime pattern (`if (res == ncclSuccess) free(w)`) is
the same class as (b): a caller that keys on the return value.

To be clear, (b) is not the usage pattern NCCL designs or documents — the 
user guide describes `ncclInProgress` as "users must query the states of the
communicators" (docs/userguide/source/api/comms.rst), and every example it
shows uses the do-while of shape (a) with a separate state variable. 

So a caller following the docs is safe today; shape (b) is a misuse that the
current return value actively invites: it hands back `ncclSuccess` while work
is still in flight, so the natural reading "success means nothing more will
happen" leads directly to the race. That is exactly what the fix below
addresses.

### Proposed fix

Make split/shrink follow the same pattern — after the group end, re-read the
parent's async state and return it:

```c
exit:
  (void)ncclGroupErrCheck(res);
  NCCLCHECK(ncclGroupEndInternal());
  if (res == ncclSuccess && !comm->config.blocking) {
    NCCLCHECK(ncclCommGetAsyncError(comm, &res));
  }
```

With this, the caller flow is uniform: accept `ncclSuccess`/`ncclInProgress`
from the call, poll the parent until it reports neither, then use the child —
`*newcomm` is valid by then, since the background thread writes it before
stamping the parent's final result.

I have this fix prepared and would be happy to open a PR if you agree with the
direction. I'd bundle the documentation update in the same PR — the
`ncclCommSplit`/`ncclCommShrink` sections currently say nothing about
non-blocking semantics (no word on polling the parent, or that `*newcomm`
stays `NCCL_COMM_NULL` until the background init completes), in contrast to
`ncclCommFinalize`/`ncclCommRevoke` whose header comments and user guide
sections explicitly describe the `ncclInProgress` state transition — updating
those would fix the doc gap independently of the code question.

