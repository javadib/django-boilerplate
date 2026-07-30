# django-copier-template

A [Copier](https://copier.readthedocs.io/) template for bootstrapping Django projects
with a consistent, configurable stack.

## Usage

```bash
pip install copier
copier copy gh:YOUR_GH_USER/django-copier-template my-new-project
```

Or from a local path:

```bash
copier copy /path/to/django-copier-template my-new-project
```

Copier will ask a series of questions (DRF, SimpleJWT, OpenAPI, database backend,
Docker, tests, CI, semantic-release, ...) and generate a ready-to-run project.

## Updating a generated project later

Since Copier tracks answers in `.copier-answers.yml`, you can pull in template
improvements later with:

```bash
cd my-existing-project
copier update
```

## Template structure

- `copier.yml` – all questions/toggles
- `template/{{project_slug}}/` – the actual project skeleton (Jinja2-templated)
