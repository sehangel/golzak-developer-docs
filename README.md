# Golzak developer documentation

The source of [developers.golzak.com](https://developers.golzak.com): the guides of the **Golzak Domains API**
(`docs/`) and its OpenAPI contract (`openapi/golzak-registrar-v1.yaml`).

This repository is **generated**: Golzak's private API repository publishes its public documentation here on every
change. Read the rendered site at [developers.golzak.com](https://developers.golzak.com).

## Suggesting a change

Use the **Edit page** link at the bottom of any page of the site, or open a pull request or an issue here. Golzak
reviews it and carries the change into the API repository, from which this one is regenerated (so a pull request is
closed once its change has arrived through the next sync, rather than merged here).

## Contents

- `docs/`: the guides, one Markdown file per page, and `llms.txt` for AI agents.
- `openapi/`: the API contract (OpenAPI 3.1). The site also serves it as `openapi.yaml` next to each version.
