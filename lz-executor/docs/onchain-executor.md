# LayerZero V2 — Nhiệm vụ On-chain của Executor

> Nguồn: https://docs.layerzero.network/v2/workers/off-chain/build-executors
> Tham chiếu code: `lz-executor/zexecutor/contracts/src/SimpleExecutor.sol`

Executor là worker bên thứ ba: nhận phí ở src chain, rồi chốt verify + chạy message ở dst chain. Phần on-chain gồm 1 interface bắt buộc và 2 hành động on-chain (commit, execute) do off-chain điều khiển.

---

## 1. `ILayerZeroExecutor` interface

Contract executor phải deploy trên **mọi** chain hỗ trợ và implement 2 hàm. Cả 2 được gọi ở **src chain** lúc gửi.

| Hàm | Loại | Khi gọi | Nhiệm vụ |
|-----|------|---------|----------|
| `assignJob(uint32 dstEid, address sender, uint256 calldataSize, bytes options) → uint256 price` | payable | Trong `_lzSend` (SendLib gọi) | Giao job cho executor + trả giá phí. Đây là lúc executor "nhận việc" và thu phí. |
| `getFee(uint32 dstEid, address sender, uint256 calldataSize, bytes options) → uint256 price` | view | App gọi trước khi gửi | Quote phí để estimate, không đổi state. |

Tham số:
- `dstEid` — endpoint id chain đích.
- `sender` — contract gửi ở src (có thể áp giá phân biệt theo sender).
- `calldataSize` — kích thước data message + params.
- `options` — tham số thêm (ví dụ gửi dust token ở dst).

Ví dụ `SimpleExecutor`: `assignJob` chỉ gọi lại `getFee`; giá = static `messageFees[dstEid]` (không dynamic theo gas). Owner set giá qua `setMessageFee`, rút phí qua `withdrawFee`.

---

## 2. Committer — chốt verification lên dst

Không phải contract riêng. Là off-chain logic, hành động on-chain = `commitVerification`. Phối hợp kết quả DVN attestation.

Luồng:
1. **Idempotency check**: gọi `ULN.verifiable(bytes _packetHeader, bytes32 _payloadHash)` → trả về verification state.
2. **Xử lý theo state**:
   - `Verifying` — chưa đủ chữ ký DVN; đợi thêm event `PayloadVerified`.
   - `Verifiable` — gọi `commitVerification(bytes _packetHeader, bytes32 _payloadHash)` để chốt verify vào ReceiveUln.
   - `Verified` — đã commit rồi; kết thúc.
3. **Kết thúc**: idempotency check lại, xác nhận state là `Verified`.

Commit mở đường cho Executor: chưa commit thì message không executable.

---

## 3. Executor — chạy message ở dst

Off-chain logic, hành động on-chain = `lzReceive`. Chạy sau khi Committer commit xong.

Luồng:
1. **Executable check**: gọi `endpoint.executable(bytes _packetHeader, bytes32 _payloadHash)` → trả execution state.
2. **Xử lý theo state**:
   - `NotExecutable` — đợi committer hoặc các nonce trước.
   - `Executable` — decode `options`, gọi `endpoint.lzReceive(Origin _origin, address _receiver, bytes32 _guid, bytes _message, bytes _extraData)` để chạy message thật (executor trả gas).
   - `Executed` — xong; kết thúc.
3. **Kết thúc**: idempotency check, xác nhận state là `Executed`.

---

## Tóm flow

```
src chain:  assignJob   (thu phí, nhận job)
   │ emit PacketSent + ExecutorFeePaid
   ▼
dst chain:  commitVerification   (Committer — khi ULN.verifiable == Verifiable)
   │ state → executable
   ▼
dst chain:  endpoint.lzReceive   (Executor — khi executable == Executable)
```

**Idempotency** ở mọi bước: event fire nhiều lần, phải tránh commit/execute trùng. Luôn check state trước khi gửi tx.
