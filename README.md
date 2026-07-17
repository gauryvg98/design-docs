# Design Docs

**Rendered live on my portfolio:** [portfolio-gauryvg98s-projects.vercel.app/#docs](https://portfolio-gauryvg98s-projects.vercel.app/#docs)

A running collection of architecture write-ups and diagrams for systems I've designed and
built — backend, blockchain, and event-driven infrastructure. Each folder is a
self-contained topic with its own diagrams and notes.

— **Yashvardhan Gaur** · [github.com/gauryvg98](https://github.com/gauryvg98)

---

## Contents

| Topic | What it covers |
|-------|----------------|
| [Multi-chain MPC custody platform](mpc-custody-platform/) | Event-driven architecture for a production BTC/ETH/TRON custody platform — indexer→wallet reconciliation, reorg safety, the MPC transfer pipeline, signed webhooks, and real-time WebSockets. Redis Streams + Pub/Sub on PostgreSQL. |
| [ChadWallet — Solana trading app](chadwallet-solana-trading/) | Non-custodial real-time Solana meme-coin trading app — cache-first poller-driven backend, a websocket hub that fans one upstream fetch out to all viewers, and a browser-signed (Privy MPC) swap pipeline. Go + Next.js, no database. |
| [Sentinel — execution-integrity-first OMS](sentinel-oms/) | Order management system for automated crypto trading — an append-only event ledger with a pure state machine, rebuildable projections, and a reconciler that halts on genuine ledger-vs-broker divergence ("halt, don't absorb"). A fleet of per-symbol bots on one account, risk sized to the stop. Python / asyncio + PostgreSQL. |

<!-- Add new topics as siblings: a folder with its own README.md + diagrams/ -->

---

## Conventions

- Diagrams are authored in [Mermaid](https://mermaid.js.org/) (`.mmd`) with `.svg` exports
  committed alongside, so they render both on GitHub and anywhere that needs a static image.
- Each topic folder is self-contained: `README.md` + a `diagrams/` directory.
- Content is generalized and anonymized — it describes engineering patterns, not any
  employer's proprietary internals.

## License

[CC BY 4.0](LICENSE) — reuse with attribution.
