# 07: Cancel orders end to end

**What to build:** A trader cancels a working order and sees it end `CANCELED` with its remaining reservation released. `CancelOrder` moves the order to `PENDING_CANCEL` and writes an `order.cancel_requested` outbox event; the gateway sends an OrderCancelRequest (35=F, OrigClOrdID = order id); the simulator cancels if leaves remain or answers OrderCancelReject otherwise; the OMS applies the cancel or reverts to the previous status on a reject. Cancelling a terminal order is refused with `FailedPrecondition`. The kill switch blocks submits but still allows cancels.

**Blocked by:** 06 (Apply execution reports and fill orders end to end).

**Status:** ready-for-agent

- [ ] `CancelOrder` on a working order returns `PENDING_CANCEL` and emits `order.cancel_requested` through the outbox
- [ ] `CancelOrder` on a terminal order returns `FailedPrecondition`
- [ ] Gateway sends 35=F with OrigClOrdID and a distinct ClOrdID
- [ ] Simulator responds ExecType=Canceled when leaves remain, OrderCancelReject otherwise
- [ ] OMS applies canceled (releases reservation) or reverts `PENDING_CANCEL` on cancel reject; state machine table tests extended
- [ ] End-to-end tests pass: cancel of a working order ends `CANCELED`; cancel of a filled order refused; kill switch rejects submits and allows cancels
