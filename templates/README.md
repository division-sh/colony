# Templates

Complete, end-to-end flows you **fork as a starting point**. Each is a self-contained flow package plus `template.yaml`, `README.md`, and `scenarios/`.

Use one by copying the directory into your project and adapting it — read the entry's `README.md` for its inputs/outputs and the backends/tools/credentials it requires.

See [../docs/authoring-guide.md](../docs/authoring-guide.md) for the layout and metadata schema, and [../CONTRIBUTING.md](../CONTRIBUTING.md) for the bar.

### Planned

- **`twitter-prospecting`** — flagship. X/Twitter account-prospecting pipeline (discovery → dedup → enrich → classify → bucket). Added once rebuilt on the batch-agent (#1447) and logic-node work, verified against fresh `platform-spec`.
