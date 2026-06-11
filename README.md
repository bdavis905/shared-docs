# Shared Docs

Public working documents for collaboration, organized by client.

**Live site:** https://bdavis905.github.io/shared-docs/

## Structure

One folder per client; each doc is a self-contained page in its own subfolder.

```
genesis/        Luke / Genesis
  copy-pipeline/        ← interactive map of the copy pipeline
  genesis-vs-exodus/    ← member explainer (keys, billing, two-systems model)
matt-beard/     Matt Beard
dlc/            DLC
```

## Documents

| Client | Doc | What it is |
|---|---|---|
| Genesis | [Genesis vs. Exodus](https://bdavis905.github.io/shared-docs/genesis/genesis-vs-exodus/) | Member explainer: one engine two drivers, every key, billing map, this week's FAQ. Pairs with exodus-v2026.6.1100. |
| Genesis | [The copy pipeline, as it runs today](https://bdavis905.github.io/shared-docs/genesis/copy-pipeline/) | Step-by-step map of the current copy flow. Technical companion: [`genesis/copy-pipeline/TECHNICAL-DETAIL.md`](genesis/copy-pipeline/TECHNICAL-DETAIL.md). |

## Adding a doc

1. Create `<client>/<doc-name>/index.html` (self-contained — inline CSS/JS).
2. Add a card for it on the root `index.html` under the client's section.
3. Add a row to the table above.
4. Push to `main` — GitHub Pages updates in about a minute.

This repo is **public**: nothing secret goes in here (no keys, no customer data, no prompts).
