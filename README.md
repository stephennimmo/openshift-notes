# openshift-notes

Documentation site for OpenShift installation and configuration, built with [Material for MkDocs](https://squidfunk.github.io/mkdocs-material).

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Development

```bash
source .venv/bin/activate
mkdocs serve --livereload
```

The site will be available at [http://localhost:8000](http://localhost:8000).

## Build

```bash
mkdocs build
```

## Deploy

The site is deployed to GitHub Pages using GitHub Actions.
