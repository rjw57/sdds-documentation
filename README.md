# Secure Data Donation Service documentation

This repository holds documentation related to the Secure Data Donation Service (SDDS) technical
stack as a whole. Documentation solely related to a single repository can be found in per-repository
documentation sources.

## Other sources of documentation

* [SDDS technical wiki][wiki]

[wiki]: https://uoy.atlassian.net/wiki/spaces/SDDSTECH

## Quickstart

Clone the repository locally. Ensure that the [uv tool][uv] and [pre-commit][pre-commit] tools
are installed.

[uv]: https://docs.astral.sh/uv/getting-started/installation/
[pre-commit]:https://pre-commit.com/

Enable pre-commit hooks:

```console
pre-commit install
```

Install dependencies:

```console
uv sync
```

Start serving documentation:

```console
uv run zensical serve
```
