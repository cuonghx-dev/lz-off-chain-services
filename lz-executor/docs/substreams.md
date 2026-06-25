# Substreams — Stack & kiến trúc

Substreams = tech index blockchain. Extract data từ chain → transform custom (Rust/WASM) → sink ra đích chọn (DB, app, file).

Data chảy 1 chiều qua 4 tầng:

```
┌──────────────────┐
│ Blockchain node  │   geth / reth / solana validator...
└────────┬─────────┘
         │ instrumented (node patch tuôn raw data)
         ▼
┌──────────────────┐
│ Firehose         │   extract raw block → protobuf, lưu flat file
└────────┬─────────┘
         │ gRPC stream Block (sf.ethereum.type.v2.Block)
         ▼
┌──────────────────┐
│ Substreams       │   transform: map/store module Rust/WASM
└────────┬─────────┘
         │ sink
         ▼
┌──────────────────┐
│ DB / app / file  │   PostgreSQL, subgraph, webhook, stream...
└──────────────────┘
```

---

## 1. Blockchain node

Node thường (geth, reth, Solana validator...) nhưng được **instrument** (patch) để tuôn toàn bộ data thực thi block ra ngoài — không chỉ JSON-RPC mà cả trace, balance change, internal call.

---

## 2. Firehose

Tầng **extract** — nền móng.

- Nhận data từ node instrumented → đóng gói thành **protobuf block** (vd `sf.ethereum.type.v2.Block`).
- Lưu block lịch sử ra **flat file** trên object storage → đọc lại nhanh, đọc song song (parallel).
- Expose qua **gRPC stream**: client subscribe nhận từng block.

Nặng, chạy gần node. Dev **không cần tự cài** — StreamingFast host public endpoint.

---

## 3. Substreams

Tầng **transform** — chạy trên Firehose.

- Input: block protobuf từ Firehose.
- Chạy **module** Rust compile sang **WASM**:
  - `map` — stateless: extract / filter / transform.
  - `store` — stateful: aggregate, persist state qua block.
- Module nối thành **DAG** (đồ thị có hướng, không vòng).
- **Parallel execution**: chia segment (default 25K block) → N worker chạy song song → backfill nhanh ~N lần.
- **Caching**: cache output theo module hash (chỉ prod mode) → run sau skip WASM.
- **Reliability**: cursor + `BlockUndoSignal` → resume chính xác khi disconnect / fork, không miss data.

---

## 4. Sink

Tầng **delivery** — đẩy output transform ra đích.

- SQL DB (PostgreSQL)
- Subgraph / The Graph
- Webhook, PubSub, stream trực tiếp app
- Flat file

Sink persist cursor → reconnect resume từ checkpoint.

---

## StreamingFast

Công ty build cả Firehose + Substreams (nay trong The Graph ecosystem). Chạy **public Firehose/Substreams endpoint** miễn phí.

**Cần Firehose để dùng Substreams?** Có — nhưng **không cần tự cài**. StreamingFast host sẵn. Dev chỉ:
1. Lấy API key.
2. Viết Substreams module.
3. Trỏ tới public endpoint → chạy.

Tự host Firehose chỉ khi: chain riêng, data privacy, hoặc scale lớn cần kiểm soát hạ tầng.

---

## Liên hệ lz-executor

Indexer hiện listen event qua RPC (xem [`design.md`](./design.md)). Substreams = lựa chọn thay thế cho tầng **Indexer**:
- Stream `PacketSent` / `ExecutorFeePaid` qua Firehose thay vì poll RPC.
- Parallel backfill lịch sử nhanh.
- Cursor/reorg handling built-in → bớt tự code logic resume + chống fork.
