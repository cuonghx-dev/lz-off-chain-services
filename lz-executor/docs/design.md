# Design — LayerZero Executor System

Hệ thống executor gồm 3 thành phần, dữ liệu chảy 1 chiều:

```
┌─────────────────┐     ┌───────────┐     ┌──────────────────────┐
│ Executor        │     │           │     │  Committer           │
│ contract        │ ──▶ │  Indexer  │ ──▶ │     +                │
│ (on-chain)      │     │           │     │  Executor (off-chain)│
└─────────────────┘     └───────────┘     └──────────────────────┘
   src chain          listen events         dst chain actions
```

---

## 1. Executor contract (on-chain)

Deploy trên **mọi** chain hỗ trợ. Implement `ILayerZeroExecutor`.

Nhiệm vụ:
- `assignJob(dstEid, sender, calldataSize, options) → price` — SendLib gọi lúc `_lzSend`. Nhận job + thu phí.
- `getFee(...) → price` — quote phí cho app trước khi gửi.

Khi message gửi đi, src chain emit:
- `PacketSent` — có packet mới.
- `ExecutorFeePaid` — executor này đã được trả phí.

Tham chiếu: [`executor-contract.md`](./executor-contract.md), [`lz-executor.md`](./lz-executor.md).

---

## 2. Indexer (off-chain — event provisioning)

Listen liên tục các src chain, lọc event liên quan đến executor mình.

Nhiệm vụ:
- Theo dõi `PacketSent` trên mọi src chain.
- **Validate được trả phí**: với mỗi `PacketSent`, nhìn lùi log trước đó tìm `ExecutorFeePaid` cho đúng executor mình; dừng khi gặp `PacketSent` trước đó.
  - Validate cẩn thận: 1 tx có nhiều message, logic sai → execute nhầm message chưa trả phí.
  - Chống fake event.
- Decode packet (header, payloadHash, nonce, dstEid, options).
- Đẩy job hợp lệ vào hàng đợi của chain đích → chuyển cho Committer + Executor.

Output: job chuẩn hoá (per dst chain), thường qua DB / message queue để tách rời với worker.

---

## 3. Committer + Executor (off-chain — execution trên dst)

Mỗi dst chain có 1 worker xử lý queue (FIFO). Mỗi job đi qua 2 bước on-chain.

### Committer
- Idempotency check: `ULN.verifiable(packetHeader, payloadHash)`.
  - `Verifying` — chưa đủ DVN attest → đợi.
  - `Verifiable` — gọi `commitVerification(packetHeader, payloadHash)`.
  - `Verified` — đã commit, sang bước execute.

### Executor
- Idempotency check: `endpoint.executable(packetHeader, payloadHash)`.
  - `NotExecutable` — đợi committer / nonce trước.
  - `Executable` — decode options → `endpoint.lzReceive(origin, receiver, guid, message, extraData)` (trả gas).
  - `Executed` — xong, pop khỏi queue.

Lỗi / cần xử lý thêm → push lại cuối queue retry.

---

## Nguyên tắc thiết kế

- **Tách role**: Indexer (provision) và Committer+Executor (execution) tách riêng, giao tiếp qua DB / message bus. Production nên là 2 app riêng để scale độc lập.
- **Idempotency mọi bước**: event fire lặp; luôn check state on-chain trước khi gửi tx, tránh commit/execute trùng.
- **State on-chain là nguồn chân lý**: dùng view contract (`ReceiveUln302View`, `EndpointV2View`) để đọc trạng thái message.

## Hướng production (cải tiến so với ví dụ tối giản)

- Dynamic fee theo gas params (không static).
- Multi-wallet + nonce management → scale + high availability.
- DB lưu message + execution state → resiliency, recovery sau crash.
- Fetch historical events lúc khởi động → recovery.
- Monitoring gas balance ví executor, xử lý tx failure.
