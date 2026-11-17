# dbguard documentation

A Sphinx + Read the Docs (`sphinx_rtd_theme`) documentation site for **dbguard** -
the eBPF-based, identity-aware database firewall and query auditor for PostgreSQL & MySQL.

## Build locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
make html          # or: sphinx-build -b html . _build/html
```

Open `_build/html/index.html`.

## Already-built HTML

A pre-built copy of the site is in `_build/html/` - open `_build/html/index.html`
directly in a browser, no toolchain needed.

## Host it

- **Read the Docs:** push this folder to a repo, import the project on readthedocs.org.
  `requirements.txt` and `conf.py` are already set up.
- **GitHub Pages:** build with `make html`, then publish `_build/html/` (e.g. the
  `gh-pages` branch, or a `peaceful_pie`/`actions` workflow that runs `sphinx-build`).

## Structure

```
conf.py              theme + extensions
index.rst            landing page + toctrees
guides/              introduction, installation, quickstart, configuration
architecture/        overview, dataflow, enforcement, cilium
features/            policy DSL, SQL firewall, PII, rate limiting, identity, audit
deployment/          single-host, kubernetes
reference/           CLI, policy schema, config reference, troubleshooting
development/          building, changelog
_static/custom.css   dark-teal accent styling
```
