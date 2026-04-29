# Polymarket Dataset

Tick-level orderbook, trade, and on-chain data from the world's largest prediction market.

Collected and maintained by [Rocklabs](https://rocklabs.io).

## Overview

| | |
|---|---|
| **Total Records** | 34B+ |
| **Markets Covered** | 220K+ |
| **On-chain History** | 2020 – Present |
| **Tick-level History** | Feb 2026 – Present |

## Datasets

### 1. CLOB Orderbook Events

Full tick-level orderbook updates — book snapshots, price changes, and cancellations across all active markets.

- **Rows:** 31.6B+
- **Coverage:** Jan 2026 – Present, 221K markets
- **Update Frequency:** Continuously updated
- **Format:** JSONL (zstd compressed), partitioned by UTC date and hour

**Schema:**

| Field | Type | Description |
|---|---|---|
| `timestamp` | string | ISO 8601 receive timestamp |
| `message_type` | string | Record type, usually `feed_message` for CLOB websocket data |
| `content` | string or object | For `feed_message`, a nested JSON string from the websocket; for metadata records, a JSON object |

For `message_type: "feed_message"`, parse `content` as JSON. The parsed payload can be a single object or an array of objects. Non-JSON heartbeat payloads such as `PONG` can be ignored.

**Orderbook `book` payload fields:**

| Field | Type | Description |
|---|---|---|
| `market` | string | Market contract address |
| `asset_id` | string | Token / outcome ID |
| `event_type` | string | `book` |
| `timestamp` | string | Exchange timestamp (unix ms) |
| `bids[]` | array | Bid price levels |
| `asks[]` | array | Ask price levels |

Each `bids` / `asks` entry:

| Field | Type | Description |
|---|---|---|
| `price` | string | Price level |
| `size` | string | Resting size at that price level |

**Orderbook `price_change` payload fields:**

| Field | Type | Description |
|---|---|---|
| `market` | string | Market contract address |
| `event_type` | string | `book` \| `price_change` |
| `timestamp` | string | Exchange timestamp (unix ms) |
| `price_changes[]` | array | Array of price level updates |

Each `price_changes` entry:

| Field | Type | Description |
|---|---|---|
| `asset_id` | string | Token / outcome ID |
| `price` | string | Price level |
| `size` | string | Size at price level |
| `side` | string | `BUY` for bid-side updates, `SELL` for ask-side updates |
| `hash` | string | Order hash |
| `best_bid` | string | Current best bid |
| `best_ask` | string | Current best ask |
| `sequence_id` / `seq_id` | string or int | Optional exchange sequence identifier |

A `size` of `0` means the price level should be removed. To reconstruct a full book, start from the latest `book` snapshot for an `asset_id`, then apply subsequent `price_change` updates in file order. If only top-of-book is needed, `best_bid` and `best_ask` on `price_change` entries can be used directly when present.

**Sample record:**

```json
{
  "timestamp": "2026-02-28T00:00:00.000Z",
  "message_type": "feed_message",
  "content": "{\"market\":\"0x8f48...c468\",\"price_changes\":[{\"asset_id\":\"15052...\",\"price\":\"0.157\",\"size\":\"55\",\"side\":\"BUY\",\"hash\":\"cbc7b3...\",\"best_bid\":\"0.157\",\"best_ask\":\"0.179\"},{\"asset_id\":\"51790...\",\"price\":\"0.843\",\"size\":\"55\",\"side\":\"SELL\",\"hash\":\"79fd9d...\",\"best_bid\":\"0.821\",\"best_ask\":\"0.843\"}],\"timestamp\":\"1772236799992\",\"event_type\":\"price_change\"}"
}
```

### 2. CLOB Trades

Individual trade fills on the Polymarket CLOB with exchange timestamps and taker attribution.

- **Rows:** 45M+
- **Coverage:** Feb 2026 – Present, 167K markets

Trade events are included in the same CLOB data files, distinguished by `event_type: "last_trade_price"` in the content payload.

**Trade payload fields:**

| Field | Type | Description |
|---|---|---|
| `asset_id` | string | Token / outcome ID |
| `event_type` | string | `last_trade_price` |
| `timestamp` | string | Exchange timestamp (unix ms) |
| `price` | string | Trade price |
| `size` | string | Trade size |
| `side` | string | Trade side as reported by the CLOB websocket |

### 3. On-chain Fills

Settlement and fill events from the CTF Exchange contract on Polygon, with maker/taker addresses and full order details.

- **Rows:** 882M+
- **Coverage:** Nov 2022 – Present, 601K token IDs
- **Format:** JSONL (zstd compressed), partitioned by date and hour

**Schema:**

| Field | Type | Description |
|---|---|---|
| `timestamp` | string | ISO 8601 timestamp |
| `message_type` | string | Event type (e.g. `onchain.OrderFilled`, `onchain.FeeCharged`) |
| `content` | object | Decoded event data |

**Content fields (OrderFilled):**

| Field | Type | Description |
|---|---|---|
| `block_number` | int | Polygon block number |
| `tx_hash` | string | Transaction hash |
| `chain_id` | int | Chain ID (137 = Polygon) |
| `contract_name` | string | `ctf_exchange` |
| `event_name` | string | `OrderFilled` |
| `decoded.maker` | string | Maker address |
| `decoded.taker` | string | Taker address |
| `decoded.makerAssetId` | string | Maker asset (token ID or `0` for USDC) |
| `decoded.takerAssetId` | string | Taker asset (token ID or `0` for USDC) |
| `decoded.makerAmountFilled` | string | Maker amount (6 decimals for USDC) |
| `decoded.takerAmountFilled` | string | Taker amount |
| `decoded.fee` | string | Fee amount |
| `ts_block` | int | Block timestamp (unix seconds) |
| `ts_recv_ms` | int | Receive timestamp (unix ms) |

**Sample record:**

```json
{
  "timestamp": "2026-02-28T00:00:10.296Z",
  "message_type": "onchain.OrderFilled",
  "content": {
    "block_number": 83558320,
    "tx_hash": "0x00ef5fc2...0bf52",
    "chain_id": 137,
    "contract_name": "ctf_exchange",
    "event_name": "OrderFilled",
    "decoded": {
      "maker": "0x030c...537b",
      "taker": "0x0829...001c",
      "makerAmountFilled": "154000",
      "takerAmountFilled": "350000",
      "makerAssetId": "0",
      "takerAssetId": "8629673...719632",
      "fee": "35000",
      "orderHash": "0x8d1c91..."
    },
    "ts_block": 1772236671,
    "ts_recv_ms": 1772236810296
  }
}
```

### 4. On-chain Events

All decoded smart contract events — CTF Exchange, NegRisk adapters, condition modules, and more.

- **Rows:** 1.5B+
- **Coverage:** Sep 2020 – Present

### 5. Positions

Position lifecycle events — splits, merges, redemptions, and payouts with per-account tracking.

- **Rows:** 438M+
- **Coverage:** Nov 2022 – Present

### 6. Market & Event Metadata

Market parameters, outcome definitions, event descriptions, and category tags for all Polymarket events and markets.

- **Rows:** 5M+

Metadata is available in two places:

- Inline raw records with `message_type: "token_index"`, `message_type: "market_metadata"`, and `message_type: "event_metadata"`.
- Daily index snapshots under `raw/_index/YYYY-MM-DD/polymarket_index.json`.

**Schema:**

| Field | Type | Description |
|---|---|---|
| `token_id` | string | Condition token ID |
| `event_slug` | string | Human-readable event slug |
| `payload` | JSON | Market details (title, description, outcomes, dates, etc.) |

Daily `polymarket_index.json` snapshots contain:

| Field | Type | Description |
|---|---|---|
| `version` | int | Snapshot schema version |
| `date` | string | UTC snapshot date |
| `generated_at` | string | ISO 8601 generation timestamp |
| `token_index` | object | `token_id -> event_slug` mapping |
| `event_index` | object | `event_slug -> compact event metadata` mapping |
| `market_index` | object | `token_id -> compact market metadata` mapping |

Use `market_index[token_id].token_outcomes` to map token IDs back to human-readable outcomes when available.

## Parsing CLOB Raw Files

Each `raw/YYYY-MM-DD/HHMM.jsonl.zst` file is an hourly UTC segment. The file is zstd-compressed JSONL. Each line is independent, so files can be streamed without loading the whole segment into memory.

Minimal parsing flow:

1. Decompress the `.jsonl.zst` stream.
2. Parse each line as JSON.
3. Keep `message_type: "feed_message"` for CLOB orderbook and trade data.
4. Parse the nested `content` string as JSON. If it is an array, process each item.
5. Route by `event_type`: `book`, `price_change`, or `last_trade_price`.
6. Join `asset_id` to market metadata using `raw/_index/YYYY-MM-DD/polymarket_index.json` or inline metadata records.

Minimal Python example:

```python
import io
import json
import zstandard as zstd

path = "raw/2026-02-06/0000.jsonl.zst"

with open(path, "rb") as fh:
    stream = zstd.ZstdDecompressor().stream_reader(fh)
    text = io.TextIOWrapper(stream, encoding="utf-8")
    for line in text:
        row = json.loads(line)
        if row["message_type"] != "feed_message":
            continue

        try:
            payload = json.loads(row["content"])
        except json.JSONDecodeError:
            continue

        events = payload if isinstance(payload, list) else [payload]
        for event in events:
            event_type = event.get("event_type")
            if event_type == "price_change":
                for change in event.get("price_changes", []):
                    token_id = change["asset_id"]
                    price = float(change["price"])
                    size = float(change["size"])
                    side = "BID" if change.get("side") == "BUY" else "ASK"
            elif event_type == "book":
                token_id = event["asset_id"]
                bids = [(float(x["price"]), float(x["size"])) for x in event.get("bids", [])]
                asks = [(float(x["price"]), float(x["size"])) for x in event.get("asks", [])]
            elif event_type == "last_trade_price":
                token_id = event["asset_id"]
                price = float(event["price"])
                size = float(event["size"])
```

## Storage Layout

Data is stored in Cloudflare R2 and organized as follows:

```
poly-raw/
├── raw/
│   ├── YYYY-MM-DD/           # CLOB orderbook & trade data
│   │   ├── 0000.jsonl.zst    # Hour 00:00 UTC
│   │   ├── 0100.jsonl.zst    # Hour 01:00 UTC
│   │   └── ...               # ~2-3 GB per hour (compressed)
│   ├── onchain/
│   │   └── YYYY-MM-DD/       # On-chain events
│   │       ├── 0000.jsonl.zst
│   │       └── ...           # ~70-100 MB per hour (compressed)
│   ├── _index/
│   │   └── YYYY-MM-DD/
│   │       ├── polymarket_index.json   # Daily market metadata index
│   │       └── onchain_index.json      # Daily on-chain address index
└── state/
    └── polygon_cursor.json   # Indexer cursor state
```

All data files use **JSONL** format compressed with **zstd**. Each line is a self-contained JSON record.

For CLOB-only access, the required paths are:

```
raw/YYYY-MM-DD/*.jsonl.zst
raw/_index/YYYY-MM-DD/polymarket_index.json
```

The following paths are separate datasets and are not required to parse CLOB orderbook data:

```
raw/onchain/
raw/external/
state/
```

## Access

We provide **free access** to these datasets for students and academic researchers.

To request access, contact us with:
- Your name and institutional affiliation
- Intended use case or research topic

## Citation

If you use this data in a publication, please cite:

```
Rocklabs Polymarket Dataset, https://rocklabs.io
```

## Contact

- **Email:** hello@rocklabs.io
- **Website:** [rocklabs.io](https://rocklabs.io)
- **Data Catalog:** [rocklabs.io/data](https://rocklabs.io/data)

## License

This dataset is provided for academic and research purposes. See [LICENSE](LICENSE) for details.
