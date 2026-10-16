---
description: >-
  Every way to submit a signed transaction to the BSV blockchain, how they
  compare, and which one to pick.
---

# Broadcasting

Broadcasting is the step that takes a fully signed transaction from your application and hands it to the network so that nodes can validate it, hold it as unconfirmed, and mine it into a block. Until a transaction has been broadcast, it exists only on your machine.

This section documents every supported way to broadcast on BSV, from the BSV Association's recommended service to a paste-and-send web form, and explains when to use each.

{% hint style="success" %}
**Recommended path:** [Arcade](arcade.md) is the BSV Association's supported transaction broadcaster. It accepts transactions over a simple HTTP API, delivers them directly to Teranode, tracks their status through to mining, and returns the merkle path you need for SPV. It replaces both mAPI and ARC. Public endpoints are hosted by the BSV Association and by GorillaPool, and you can also run your own.
{% endhint %}

## Before you broadcast

Every method on this page expects the same input: a complete, signed transaction serialised to bytes. You will usually have it in one of three encodings.

| Encoding | What it carries | Where it is accepted |
| --- | --- | --- |
| **Raw** | The transaction exactly as it appears on chain. | Everywhere. |
| **Extended Format (EF)** | Raw transaction plus, for each input, the satoshis and locking script of the output being spent ([BRC-30](https://bsv.brc.dev/transactions/0030)). Lets the receiver validate fees and scripts without a database lookup. | Arcade, ARC, Teranode. |
| **BEEF** | The transaction plus its unconfirmed ancestors and their merkle paths ([BRC-62](https://bsv.brc.dev/transactions/0062)). Designed for peer-to-peer exchange between wallets. | ARC only. Arcade and node RPCs reject it. Convert to EF before submitting. |

The SDKs produce EF automatically when every input has its source transaction attached, and fall back to raw otherwise. If you build transactions by hand, raw hex is accepted by every method below.

## Ways to broadcast

| Method | Interface | Input | Status tracking | Auth | Best for |
| --- | --- | --- | --- | --- | --- |
| [**Arcade**](arcade.md) | REST, SDK classes | Raw, EF | Full lifecycle, webhooks, SSE, merkle path on mining | None on public endpoints | Any application. The default choice. |
| [**API broadcasters**](api-broadcasters.md) (WhatsOnChain, Bitails, BananaBlocks) | REST | Raw hex | Separate explorer lookups | Optional API key for higher limits | Prototypes, low volume, apps already using an explorer API |
| [**Web broadcasters**](web-broadcasters.md) (WhatsOnChain, Bitails) | Browser form | Raw hex | Explorer page | None | One-off manual sends, debugging, support |
| [**Node RPC**](node-rpc.md) (SV Node, Teranode) | JSON-RPC or HTTP | Raw, EF | Mempool and chain queries on the same node | Your own credentials, or none on the public endpoint | Operators running their own node; last-resort fallback |
| [**ARC and mAPI**](legacy-arc-mapi.md) | REST | Raw, EF, BEEF | Lifecycle callbacks | API key for commercial tiers | Existing integrations not yet migrated to Arcade |

## Choosing a method

- **You are building an application or wallet.** Use Arcade. Point your SDK at a public endpoint for development, and run your own instance or use a paid provider when you need an SLA.
- **You already depend on an explorer API** for UTXO lookups or history and send a handful of transactions. Their broadcast endpoints are fine, but watch the rate limits and plan to move to Arcade as volume grows.
- **You need to push one transaction right now** and have the hex in a clipboard. Use a web broadcaster.
- **You run your own SV Node or Teranode.** Broadcast through it. You get the same validation the miners apply, and you are not dependent on a third party.
- **You have an ARC integration in production.** It still works. Arcade is API-compatible on the parts that matter, so migration is mostly a URL change. See the [legacy page](legacy-arc-mapi.md).

## What "accepted" means

A successful broadcast response means the receiver has validated the transaction and will relay it. It does not mean the transaction is mined. The transaction then moves through a lifecycle that Arcade reports explicitly and that other methods leave you to poll for:

```
RECEIVED → SENT_TO_NETWORK → ACCEPTED_BY_NETWORK → SEEN_ON_NETWORK → MINED → IMMUTABLE
                                                          ↘ DOUBLE_SPEND_ATTEMPTED / REJECTED
```

`SEEN_ON_NETWORK` is the point at which the transaction is safe to treat as a zero-confirmation payment for ordinary value. `MINED` comes with a merkle path so you can verify inclusion yourself. `IMMUTABLE` means enough blocks have built on top that a reorganisation is no longer a practical concern.

## Good practice for every method

- **Keep the raw transaction until you have seen it on the network.** If a broadcast fails with a timeout or a 5xx, resubmit the same bytes. Do not rebuild the transaction with the same inputs, or you risk a double spend against your own earlier attempt.
- **Resubmission is safe.** Arcade and node RPCs are idempotent for an identical transaction. Explorer APIs return an "already known" error you can treat as success.
- **Treat a 400 as final.** It means the transaction itself is invalid: a bad signature, a fee below policy, a non-final locktime. Sending it somewhere else will not help.
- **Check policy before building.** Minimum fee rates and maximum sizes vary by operator. Arcade exposes them at `GET /policy`.
- **Have a second route.** Configure a fallback broadcaster for when your primary is unreachable. Any of the methods on this page can serve that role, including the [public SV Node RPC endpoint](../network-topology/nodes/sv-node/rpc-interface/public-rpc-endpoints.md).
