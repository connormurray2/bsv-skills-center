---
description: >-
  The BSV Association's supported transaction broadcaster for Teranode. Public
  endpoints, API reference, SDK integration, and self-hosting.
---

# Arcade

[Arcade](https://github.com/bsv-blockchain/arcade) is the BSV Association's transaction broadcast service. It accepts a signed transaction over HTTP, validates it against network policy, delivers it to Teranode, and then follows it through the network until it is mined, reporting each status change by webhook or server-sent events and handing back the merkle path once the transaction is in a block.

Arcade is the successor to [ARC and mAPI](legacy-arc-mapi.md). Its HTTP API keeps ARC's request and response shapes wherever practical, so most ARC clients can be repointed with a URL change. Unlike ARC, it is built for Teranode only and runs as a single binary with no external database.

{% embed url="https://github.com/bsv-blockchain/arcade" %}

## Public endpoints

The following instances are open to anyone, on mainnet, with no API key.

| Operator | Region | URL |
| --- | --- | --- |
| BSV Association | United States | `https://arcade-us-1.bsvblockchain.tech` |
| BSV Association | Europe | `https://arcade-eu-1.bsvblockchain.tech` |
| BSV Association | Asia-Pacific | `https://arcade-ap-1.bsvblockchain.tech` |
| GorillaPool | — | `https://arcade.gorillapool.io` |

Each instance publishes its own fee and size limits at `GET /policy`. They differ by operator, so read the policy from the instance you intend to use rather than assuming one value across the network. Every instance also serves interactive API documentation at its root URL and a liveness report at `/health` that lists the Teranode endpoints it is connected to.

{% hint style="info" %}
The public endpoints are shared infrastructure, suitable for development and for production applications with ordinary volume. If you need guaranteed throughput, an SLA, or private callback handling, run your own instance (see [Self-hosting](#self-hosting)) or arrange a commercial agreement with an operator.
{% endhint %}

## Quick start

Submit a transaction as hex:

```bash
curl -X POST https://arcade-us-1.bsvblockchain.tech/tx \
  -H "Content-Type: text/plain" \
  --data "<hex-encoded transaction>"
```

```json
{ "txid": "9f2b…", "status": 202, "txStatus": "RECEIVED" }
```

Check its status:

```bash
curl https://arcade-us-1.bsvblockchain.tech/tx/<txid>
```

```json
{
  "txid": "9f2b…",
  "txStatus": "MINED",
  "timestamp": "2026-10-16T12:00:00Z",
  "blockHash": "0000…",
  "blockHeight": 971333,
  "merklePath": "<BUMP hex>"
}
```

## API reference

Every path is relative to the instance URL. The full OpenAPI description lives in the repository at [`openapi/arcade.openapi.yaml`](https://github.com/bsv-blockchain/arcade/blob/main/openapi/arcade.openapi.yaml).

### `POST /tx` — submit one transaction

Pick the body encoding that suits your client. Bodies are capped at 32 MiB.

| `Content-Type` | Body |
| --- | --- |
| `text/plain` | Hex-encoded transaction. |
| `application/octet-stream` | Raw transaction bytes. Most efficient. |
| `application/json` | `{ "rawTx": "<hex>" }`. The ARC-compatible form. |

Raw and [Extended Format](https://bsv.brc.dev/transactions/0030) are both accepted. BEEF is not; convert it to EF before submitting. Arcade runs policy, fee, and script validation synchronously before answering, so EF is preferred because it lets the validator check fees and scripts without looking up the parent outputs.

Optional headers:

| Header | Effect |
| --- | --- |
| `X-CallbackUrl` | Arcade POSTs status updates to this URL as the transaction progresses. |
| `X-CallbackToken` | Opaque token of your choosing. Sent back on every webhook and used to filter the SSE stream. |
| `X-FullStatusUpdates` | `true` to receive every intermediate status, not only the final ones. |
| `X-SkipFeeValidation`, `X-SkipScriptValidation` | Bypass the corresponding local check. Honoured only where the operator allows it. |

Responses:

| Status | Meaning |
| --- | --- |
| `202` | Accepted for processing. Body is `{ txid, status: 202, txStatus }`. Resubmitting a transaction Arcade has already seen also returns 202, with `txStatus` reflecting its current state, which may already be `MINED`, `REJECTED`, or `DOUBLE_SPEND_ATTEMPTED`. |
| `400` | The transaction failed to decode or failed validation. The body is `{ error, reason, txid, status, title, detail, type }`, where `status` is the ARC error code (461 malformed scripts, 465 fee too low, 476 not final, and so on). This is terminal: the transaction is invalid as submitted, and sending it elsewhere will not help. The one exception is 476, a locktime that has not yet passed, which you may resubmit later. |
| `503` | Arcade is under backpressure. Nothing was queued. Retry after a short delay. |

### `POST /txs` — submit a batch

Concatenate raw transaction bytes with no separators and send them as `application/octet-stream`. Bodies are capped at 256 MiB. The response reconciles the batch:

```json
{ "submitted": 42, "duplicates": 3, "total": 45 }
```

If any transaction in the batch fails validation the whole call returns `400` naming the offending txid, and nothing from that call is published.

### `GET /tx/{txid}` — status

Returns the transaction's current state. Once mined, the response includes `blockHash`, `blockHeight`, and `merklePath`, a hex-encoded [BUMP](https://bsv.brc.dev/transactions/0074) you can verify against block headers. Status records are kept indefinitely. A `404` means this instance has never issued a verdict for the txid, so if you are using several instances, query the one you submitted to.

Add `?callbackToken=<token>` to see the delivery state of your webhooks for this transaction.

### `GET /policy` — fee and size limits

Returns the policy the instance enforces at intake, in the same shape ARC used. `GET /v1/policy` is an alias.

```json
{
  "policy": {
    "miningFee": { "satoshis": 1, "bytes": 1000 },
    "maxtxsizepolicy": 100000000,
    "maxscriptsizepolicy": 100000000,
    "maxtxsigopscountspolicy": 4294967295,
    "standardFormatSupported": true
  },
  "timestamp": "2026-10-16T12:00:00Z"
}
```

Unless the operator has pinned a value, `miningFee` tracks the lowest rate currently advertised by Teranode peers, and the size limits track the most generous peer. Read it before building large or low-fee transactions.

### `GET /events` — server-sent events

A long-lived stream of status updates for every transaction submitted with a given `X-CallbackToken`:

```bash
curl -N "https://arcade-us-1.bsvblockchain.tech/events?callbackToken=<token>"
```

```
id: 1716205200123456789
event: status
data: {"txid":"9f2b…","txStatus":"SEEN_ON_NETWORK","timestamp":"2026-10-16T12:00:00.123Z"}

id: 1716205260987654321
event: status
data: {"txid":"9f2b…","txStatus":"MINED","timestamp":"2026-10-16T12:01:00.987Z","blockHash":"0000…","blockHeight":971333,"merklePath":"<BUMP hex>"}
```

A `: keepalive` comment arrives every 15 seconds. If the connection drops, reconnect with the `Last-Event-ID` header set to the last `id` you saw and Arcade replays everything you missed. CORS is open, so a browser can subscribe directly with `EventSource`.

### Webhooks

When you submit with `X-CallbackUrl`, Arcade POSTs a JSON body to that URL on each status change:

```json
{
  "timestamp": "2026-10-16T12:01:00Z",
  "txid": "9f2b…",
  "txStatus": "MINED",
  "blockHash": "0000…",
  "blockHeight": 971333,
  "merklePath": "<BUMP hex>"
}
```

Requests carry a `User-Agent` beginning `arcade-webhook/` and, if you supplied one, your callback token. Failed deliveries are retried with linear backoff (1, 2, 3 minutes and so on) up to the operator's configured limit. Callback URLs must be publicly routable; private and loopback addresses are refused at submit time.

### Transaction statuses

| Status | Meaning |
| --- | --- |
| `RECEIVED` | Validated locally and queued. |
| `SENT_TO_NETWORK` | Delivered to at least one Teranode. |
| `ACCEPTED_BY_NETWORK` | A Teranode accepted it into its mempool. |
| `SEEN_ON_NETWORK` | Observed in a miner's subtree. Safe to treat as zero-confirmation. |
| `SEEN_MULTIPLE_NODES` | Observed by more than one miner. Informational. |
| `MINED` | Included in a block. `merklePath` is now available. |
| `IMMUTABLE` | Deep enough that reorganisation is no longer a practical concern. |
| `PENDING_RETRY` | Delivery to Teranode failed; Arcade will retry. |
| `STUMP_PROCESSING` | Mined; merkle path is being assembled. |
| `REJECTED` | Rejected by the network. Terminal. |
| `DOUBLE_SPEND_ATTEMPTED` | A conflicting transaction was seen. Terminal in Arcade, unlike ARC where it could still resolve. `competingTxs` lists the conflicts. |
| `UNKNOWN` | No verdict yet. |

If a block is orphaned after a `MINED` report, Arcade re-anchors the transaction to the replacement block and emits a new `MINED` event with a fresh merkle path. The superseded anchors are kept under `orphanedProofs` on the status record.

## Using Arcade from the SDKs

### TypeScript

The `Teranode` broadcaster in `@bsv/sdk` posts a transaction as Extended Format bytes to whatever URL you give it, which is exactly what Arcade's `/tx` endpoint accepts. Every input must have its `sourceTransaction` set so the SDK can produce EF.

```ts
import { Transaction, Teranode } from '@bsv/sdk'

const arcade = new Teranode('https://arcade-us-1.bsvblockchain.tech/tx')

const tx = Transaction.fromHex(signedTxHex)
const result = await tx.broadcast(arcade)
// { status: 'success', txid: '…', message: 'broadcast successful' }
```

If you need callbacks or a callback token, implement the `Broadcaster` interface directly. The [custom broadcaster example](../guides/sdks/ts/examples/example_building_custom_tx_broadcaster.md) shows the pattern; swap the WhatsOnChain URL for Arcade's `/tx`, send `{ rawTx: tx.toHexEF() }` as JSON, and add the `X-CallbackUrl` and `X-CallbackToken` headers.

{% hint style="warning" %}
The SDK's `ARC` class posts to `{url}/v1/tx`, a path Arcade does not serve. Do not point `ARC` at an Arcade instance. The default broadcaster used by `tx.broadcast()` with no argument is still GorillaPool's ARC; pass an explicit broadcaster to use Arcade.
{% endhint %}

### Go

The Go SDK's `Arc` broadcaster posts `{ "rawTx": "<EF or raw hex>" }` to `{ApiUrl}/tx` and accepts any 2xx response, so it works against Arcade unchanged. Give it the instance root as `ApiUrl`.

```go
import (
    "github.com/bsv-blockchain/go-sdk/transaction"
    "github.com/bsv-blockchain/go-sdk/transaction/broadcaster"
)

tx, _ := transaction.NewTransactionFromHex(signedTxHex)

callbackURL := "https://your.app/arcade-webhook"
success, failure := tx.Broadcast(&broadcaster.Arc{
    ApiUrl:      "https://arcade-us-1.bsvblockchain.tech",
    CallbackUrl: &callbackURL,
})
if failure != nil {
    // failure.Code, failure.Description
}
// success.Txid
```

For ARC itself the same struct expects the versioned base, for example `https://arc.taal.com/v1`. The `WaitFor` and `WaitForStatus` fields are ARC-only and are ignored by Arcade.

### Python

The Python SDK ships a dedicated `Arcade` class.

```python
from bsv.broadcasters import Arcade, ArcadeConfig

arcade = Arcade(
    "https://arcade-us-1.bsvblockchain.tech",
    ArcadeConfig(
        callback_url="https://your.app/arcade-webhook",
        callback_token="a-token-you-choose",
    ),
)

result = await tx.broadcast(arcade)        # async
result = arcade.sync_broadcast(tx)         # sync

status = arcade.check_transaction_status(txid)
```

It submits EF when every input has its `source_transaction`, falls back to raw hex otherwise, treats a `400` as terminal, and reports `REJECTED` or `DOUBLE_SPEND_ATTEMPTED` on a 202 resubmit as a `BroadcastFailure`. The SDK's own [Arcade guide](https://github.com/bsv-blockchain/py-sdk/blob/master/docs/arcade_broadcasting.md) lists the remaining differences from its `ARC` class.

## Self-hosting

Running your own Arcade gives you private callback handling, your own rate limits, and no dependence on a third party for broadcast. The standalone profile needs only Go, about a gigabyte of disk, and network access to Teranode peers.

```bash
git clone https://github.com/bsv-blockchain/arcade.git
cd arcade
go build -o arcade ./cmd/arcade
```

Minimal `config.yaml` for mainnet:

```yaml
mode: all
log_level: info
network: mainnet
storage_path: ~/.arcade

api:
  host: 127.0.0.1
  port: 8080

health:
  port: 8081

kafka:
  backend: memory
  consumer_group: arcade

store:
  backend: pebble
  pebble:
    path: ~/.arcade/pebble

p2p:
  datahub_discovery: true

chaintracks_server:
  enabled: true

# Required for status progression beyond RECEIVED.
callback_url: "https://arcade.example.com/api/v1/merkle-service/callback"
callback_token: "<output of: openssl rand -hex 32>"

merkle_service:
  url: "https://merkle-service-us-1.bsvb.tech"
```

```bash
./arcade --config config.yaml
curl http://localhost:8080/health
```

Points to know before going to production:

- **Merkle Service is what advances status.** Arcade delegates proof construction to a separate Merkle Service, which calls back into your instance at `callback_url`. Without it every transaction stays at `RECEIVED` forever. The BSV Association hosts public Merkle Service instances for mainnet (`https://merkle-service-us-1.bsvb.tech`), testnet (`https://merkle-service-testnet-us-1.bsvb.tech`), and teratestnet (`https://merkle-service-ttn-us-1.bsvb.tech`). Your `callback_url` must be reachable from the Merkle Service, not just from your own machine.
- **Peers are discovered automatically.** With `datahub_discovery: true` Arcade finds Teranode endpoints over libp2p using built-in bootstrap peers for mainnet, testnet, and teratestnet. Regtest needs `bootstrap_peers` set by hand.
- **Bind the API privately or add auth.** The API has no authentication by default. Keep `api.host` on a private interface or put it behind an authenticating proxy if the host is internet-facing.
- **Scale by splitting services.** `mode: all` runs the API server, propagation, webhook delivery, SSE, and chain tracking in one process. Production deployments use external Kafka and Aerospike and run the services separately; start from `config.example.yaml` in the repository.
- **Every setting can be an environment variable** prefixed `ARCADE_`, for example `ARCADE_NETWORK=testnet`.

The repository's [getting started guide](https://github.com/bsv-blockchain/arcade/blob/main/docs/getting-started.md) covers per-network configuration and the alternative storage backends in more detail.
