# Executor Contract — tham chiếu

So sánh contract executor on-chain: bản tối giản (zexecutor) vs bản production của LayerZero.

## SimpleExecutor (ví dụ tối giản)

- Local: [`zexecutor/contracts/src/SimpleExecutor.sol`](../zexecutor/contracts/src/SimpleExecutor.sol)
- Upstream: https://github.com/0xpaladinsecurity/zexecutor/blob/main/contracts/src/SimpleExecutor.sol

Implement `ILayerZeroExecutor` ở mức tối thiểu:
- `assignJob(dstEid, sender, calldataSize, options) → price` — chỉ gọi lại `getFee`.
- `getFee(dstEid, ...) → price` — static `messageFees[dstEid]`, không dynamic theo gas.
- `setMessageFee` / `withdrawFee` — owner set giá + rút phí.

Mục đích: minh hoạ business logic, **không** dùng production.

## Executor.sol (LayerZero — production)

- Upstream: https://github.com/LayerZero-Labs/LayerZero-v2/blob/main/packages/layerzero-v2/evm/messagelib/contracts/Executor.sol
- Path: `packages/layerzero-v2/evm/messagelib/contracts/Executor.sol`

Bản đầy đủ, ngoài `assignJob` / `getFee` còn:
- `nativeDrop` — drop native token cho receiver, gas limit cấu hình được.
- `execute302` — execute message qua endpoint V2, có alert callback khi fail.
- `compose302` — xử lý compose message qua endpoint.
- `nativeDropAndExecute302` — gộp native drop + execute trong 1 tx.
- `execute301` / `nativeDropAndExecute301` — verify + commit cho endpoint V1.

Có ACL check, admin control, error handling đầy đủ.

## Interface chung

`ILayerZeroExecutor`:
https://github.com/LayerZero-Labs/LayerZero-v2/blob/main/packages/layerzero-v2/evm/messagelib/contracts/interfaces/ILayerZeroExecutor.sol

Chi tiết nhiệm vụ on-chain: xem [`lz-executor.md`](./lz-executor.md).
