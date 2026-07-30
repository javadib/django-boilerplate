# AGENTS.md

## Project

`django-copier-template` — a [Copier](https://copier.readthedocs.io/) template
for bootstrapping Django backend services. This is not an application; it's
the source template used to generate real projects. Everything below applies
to editing the *template itself* (files under `template/`, plus `copier.yml`),
not to code inside an already-generated downstream project.

## Prime directive: every feature is independently pluggable

The single most important rule in this repo: **any toggle in `copier.yml` can
be turned on or off, in any combination, and the generated project must still
be a valid, working Django project.** Concretely:

- Turning a feature OFF must never leave a dangling import, an empty-but-
  required settings block, a broken URL include, or a dependency that's
  referenced in code but not installed.
- Turning a feature ON must not silently require another feature unless
  that dependency is modeled explicitly via `when` in `copier.yml` (e.g.
  SimpleJWT/OpenAPI require DRF).
- The bare-minimum project (as many toggles off as their `when` clauses
  allow) must still pass `python manage.py check`, run `migrate`, and start
  with `runserver`.

Never write code that assumes a feature is present unless it's inside that
feature's own `{% if %}` guard.

## Dependent-question rule (bit us once — do not repeat)

If question B only makes sense when question A is true (`when: "{{ A }}"`),
question B's `default` must be `"{{ A }}"`, not a hardcoded `true`. `when:
false` only hides the prompt — it does **not** force the answer to false.
A hardcoded default means disabling A silently leaves B `true`, and B's code
renders into a project missing A's dependency. This exact bug happened with
`use_simplejwt`/`use_openapi` vs `use_drf`; it's fixed now — apply the same
default-mirrors-parent pattern to every new dependent question.

## Before building a new feature/module

1. Find the most similar existing feature and read how it's wired end to
   end: the `copier.yml` question, the conditional filename(s), the
   `INSTALLED_APPS` entry, the `pyproject.toml` dependency block, the
   `.env.example` entry, the README stack line. Match that pattern — don't
   invent a new convention for something that already has one.
2. Any new user-facing choice (a toggle, a provider, a version) becomes a
   Copier question in `copier.yml`. Nothing user-configurable gets
   hardcoded into a template file "for now."
3. If the feature needs a package, add it to `dependencies` or
   `[dependency-groups] dev` in `pyproject.toml.jinja`, gated by the same
   condition as the feature.

## Conditional file/directory naming convention

Copier renders the whole relative path as Jinja, so a file named
`{% if use_docker %}docker-compose.yml{% endif %}.jinja` disappears
entirely (empty rendered name = skipped) when `use_docker` is false. Every
optional file/dir in this template is gated this way — don't use `_exclude`
or post-generation deletion tasks instead; stay consistent.

## Whitespace control

`_envops` in `copier.yml` does not set `trim_blocks`/`lstrip_blocks`
(changing that now would require re-validating every existing file), so
every template file controls its own whitespace with `{%- -%}` markers.
When adding `{% if %}` blocks to an existing file, match its trim style and
actually check the rendered output — don't assume it looks fine.

## Required validation before calling anything "done"

A template change is not finished because the Jinja "looks right." Actually
regenerate and check:

```bash
pip install copier --break-system-packages
copier copy --trust --defaults . /tmp/test_default
copier copy --trust --defaults --data database=sqlite --data use_docker=false \
  --data use_drf=false --data use_celery=false . /tmp/test_minimal
```

For each profile, at minimum:

```bash
cd /tmp/test_default && ruff check . && \
  DJANGO_SECRET_KEY=x DATABASE_URL="sqlite:///$(pwd)/db.sqlite3" \
  DJANGO_SETTINGS_MODULE=config.settings.local python manage.py check
```

Test the default profile and at least one profile with your new toggle(s)
turned off. If the change touches the database, tests, or Docker layer,
also run `migrate` and `pytest` for real — not just `check`.

## Stack conventions (current defaults — keep consistent)

- Package manager: **uv**, PEP 621 `pyproject.toml`, `[dependency-groups]`
  for dev deps (not the older `[tool.uv] dev-dependencies` style).
- Settings: split into `config/settings/{base,local,production}.py`, not a
  single `settings.py`. Env vars via `django-environ`; `DATABASES` resolved
  from one `DATABASE_URL` via `env.db()` — don't hardcode per-backend
  connection params outside of the `_default_db_url` fallback.
- App layout: `apps/<name>/`, not top-level Django apps — `__init__.py`,
  `apps.py`, `models.py`, `serializers.py`/`views.py`/`urls.py` as needed.
- Linting: **ruff** only (replaces black/isort/flake8 — don't add them).
- Tests: **pytest** + **pytest-django**. `DJANGO_SETTINGS_MODULE` for tests
  is `config.settings.local`.
- Releases: **python-semantic-release**, conventional commits. Commit
  messages in this repo (and in generated projects that keep the toggle on)
  must follow `feat:` / `fix:` / `chore:` etc., since semantic-release
  parses them to pick the next version.
- Docker: multi-stage, uv-based. Any service bind-mounting the project dir
  (`.:/app`) for live-reload must also mount a named volume over
  `/app/.venv` so the image-built virtualenv isn't shadowed.
- Language: all code, comments, and commit messages are in English,
  regardless of the language used to discuss the project.

## Things intentionally out of scope for the core skeleton

Keep the base lean. Ideas like a custom user model, CORS, structured
logging, Sentry, an `api/v1/` prefix, or a real CRUD example app are
legitimate additions, but each goes through the process above — its own
toggle, its own conditional files, its own validation pass — rather than
being bolted on as always-on behavior.

## Updating already-generated projects

Downstream projects track their answers in `.copier-answers.yml` and pull
in template changes via `copier update`. Do not rename existing question
keys or repurpose their meaning — that breaks `copier update` for every
project already generated from this template. If a question needs to
change shape, add a new question and deprecate the old one instead of
mutating it in place.

## Keeping this file current

When a convention above changes (a new default tool, a new required
validation step, a new architectural rule), update this file in the same
change — it should never describe a practice the codebase has moved away
from.
