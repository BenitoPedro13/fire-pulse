# Fire Pulse

A live, public map of **fires in Brazil**, built from real public data. Satellite fire
detections (NASA FIRMS, INPE) are joined in space and time with wind and air-quality data, so
you can watch a fire get detected and then see what the air did in nearby cities in the hours
after.

It's also a portfolio case study in distributed, data-heavy real-time systems, after
Renewable Pulse, Apuração 2026 and Transit Pulse. This one is about **spatial and temporal
joins across several streams** that arrive at different rates and out of order.

**Status: planning, no code yet.** The source inventory is partly verified (2026-10-30); OpenAQ and INMET are waiting on keys.

## Docs

- [`CLAUDE.md`](CLAUDE.md): workflow rules, decisions, invariants and the stack table.
- [`docs/research/README.md`](docs/research/README.md): the verified source inventory.
- [`docs/architecture.md`](docs/architecture.md): scope, contracts, the join, hosting, replay
  and open questions.
- [`docs/tasks/`](docs/tasks/): one task document per piece of work, written before the work.
