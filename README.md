# Design Docs

A running collection of architecture write-ups and diagrams for systems I've designed and
built — backend, blockchain, and event-driven infrastructure. Each folder is a
self-contained topic with its own diagrams and notes.

— **Yashvardhan Gaur** · [github.com/gauryvg98](https://github.com/gauryvg98)

---

## Contents

| Topic | What it covers |
|-------|----------------|
| [Multi-chain MPC custody platform](mpc-custody-platform/) | Event-driven architecture for a production BTC/ETH/TRON custody platform — indexer→wallet reconciliation, reorg safety, the MPC transfer pipeline, signed webhooks, and real-time WebSockets. Redis Streams + Pub/Sub on PostgreSQL. |

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
