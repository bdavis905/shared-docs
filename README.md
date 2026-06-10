# Shared Docs

Public working documents for collaboration, organized by client.

**Live site:** https://bdavis905.github.io/shared-docs/

## Structure

One folder per client; each doc is a self-contained page in its own subfolder.

```
genesis/        Luke / Genesis
  copy-pipeline/        ← interactive map of the copy pipeline
matt-beard/     Matt Beard
dlc/            DLC
```

## Documents

| Client | Doc | What it is |
|---|---|---|
| Genesis | [The Copy Machine](https://bdavis905.github.io/shared-docs/genesis/copy-pipeline/) | Interactive map of the copy pipeline as it runs today — mark where humans should step in. Technical companion: [`genesis/copy-pipeline/TECHNICAL-DETAIL.md`](genesis/copy-pipeline/TECHNICAL-DETAIL.md). |

## Adding a doc

1. Create `<client>/<doc-name>/index.html` (self-contained — inline CSS/JS).
2. Add a card for it on the root `index.html` under the client's section.
3. Add a row to the table above.
4. Push to `main` — GitHub Pages updates in about a minute.

This repo is **public**: nothing secret goes in here (no keys, no customer data, no prompts).
