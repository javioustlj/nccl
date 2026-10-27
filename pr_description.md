<!-- PR body for github.com/NVIDIA/nccl — paste into the PR description box.
     Title: "docs: document nonblocking split/shrink/grow polling contract" -->

## Description

In nonblocking mode, `ncclCommSplit`, `ncclCommShrink` and `ncclCommGrow`
return while the new communicator's initialization is still running in the
background. The tracking communicator's state must then be polled with
`ncclCommGetAsyncError` until it reports `ncclSuccess` before the
communicators can be used again: for split and shrink that is the **parent**
communicator (and `*newcomm` is only assigned when the background init
completes), while for grow `*newcomm` is assigned at return and the state is
tracked by the **new** communicator, as with `ncclCommInitRankConfig`.

The documentation did not say any of this, and worse, the
`ncclCommGetAsyncError` section stated the opposite for these calls — that
operations which do not require a stream argument "can be considered complete
as soon as the function returns *ncclSuccess*". A reader following that
sentence concludes that a nonblocking split returning `ncclSuccess` needs no
polling — which is exactly the misuse behind #2509 (the caller freed the
storage the background initialization was still writing).

This PR is documentation only — no code change:

- **docs/userguide/source/api/comms.rst**:
  - Fix the `ncclCommGetAsyncError` section: a return code alone (including
    `ncclSuccess`) does not prove that no NCCL work remains in flight;
    callers must poll the tracking communicator until it reports
    `ncclSuccess`, and the section now names which communicator is the
    tracking one for nonblocking split/shrink/grow.
  - Add a short nonblocking paragraph to the `ncclCommSplit`,
    `ncclCommShrink` and `ncclCommGrow` sections stating which
    communicator tracks the operation and when `*newcomm` is assigned.
- **docs/userguide/source/usage/communicators.rst**: add a nonblocking
  split example to the *Creating more communicators* section showing the
  required pattern — unconditionally poll the parent after the call, do
  not key on the return value; also add an anchor to the existing
  *Error handling and communicator abort* section so it can be referenced.

## Related Issues

Documentation gap identified in #2509. Complements the code change in #2514
(the polling contract holds today; #2514 makes the return value consistent
with it).

## Changes & Impact

Documentation only. No code, no behavior change, no ABI change.

## Performance Impact

None.


