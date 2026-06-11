# django-js-asset — agent notes

`JS`, `CSS`, `JSON` support for `django.forms.Media`, plus importmap and CSP
nonce support.

## Running tests

Use **tox** (configured with `tox-uv`, so it provisions the right Python/Django
automatically — do not hand-roll a venv):

```bash
tox -e py312-dj42      # lowest supported combination
tox -e py314-dj62      # a recent combination
tox run-parallel       # the whole matrix
```

The matrix lives in `tox.ini` (`tests/manage.py test testapp`).

## Compatibility (hard constraint)

- Python `>=3.10`, Django `>=4.2`. **Django 6.2+ is NOT an acceptable floor.**
- `JS`/`CSS` *produce* Django's own `Script`/`Stylesheet` (see Layout). Where
  Django lacks them they are **backported** in `js_asset/_compat.py`, so the
  4.2 floor holds — we provide the machinery, we don't *depend* on Django
  having it. Availability in real Django: `Script` (5.2+), `Stylesheet` +
  `MediaAsset.render(attrs=)` (6.1+), attribute-aware `__eq__`/`__hash__`
  (6.2+). The `<5.2` backport mirrors the **6.2** contract.
- Don't rely on Django's `MediaAsset.render(attrs=...)` existing: it is absent
  on 5.2/6.0. `media.py:_render_asset` injects the nonce itself by rebuilding
  the tag from `element_template` + `flatatt`, which works on every version.
- Cross-version gotcha: Django >= 6.2 wraps bare js/css path strings into
  `Script`/`Stylesheet` in `Media._js`/`._css`; older Django keeps raw strings.
  `media.py:_render_{js,css}` wrap any leftover strings via `JS()`/`CSS()`, so
  `_render_asset` always sees a `MediaAsset` (or `JSON`/`ImportMap`).

## Layout

- `js_asset/_compat.py` — `MediaAsset`/`Script`/`Stylesheet`: imported from
  Django where present, backported (with the 6.2 contract) below 5.2/6.1.
- `js_asset/js.py` — `JS`/`CSS` are **factories** (a `_ProducesAsset`
  metaclass): calling them returns a Django `Script`/`Stylesheet`/`InlineStyle`
  so they dedup in `forms.Media.merge` against native assets *and* bare path
  strings; `isinstance(x, JS)` still works via `__instancecheck__`. `JSON` and
  `ImportMap` have no Django counterpart and stay standalone `@html_safe`
  objects with `render(*, nonce="")`. Output is byte-identical to native Django
  assets (flatatt sorts attributes), which the exact-string tests depend on.
  Equality is Django's, so dedup is attribute-aware on 4.2-5.1 + 6.2+ and
  path-only on 5.2-6.1 (`test_set` derives its expectation from this).
- `js_asset/media.py` — `Media(forms.Media)` subclass: merges embedded
  `ImportMap`s into one tag and applies a nonce. Implements `__add__` **and**
  `__radd__` so it keeps its type (and nonce) when combined with plain
  `forms.Media` from either side. The nonce lives on the instance (constructor
  `nonce=` or `with_nonce()` returning a copy); `render()` reads it, since
  templates call `render()` with no arguments.
- Django >= 6.2 has built-in CSP support: the `{% csp_nonce_attr media %}` tag
  (`django.utils.csp.nonce_attr`) renders media via
  `media.render(attrs={"nonce": nonce})`. `Media.render()` therefore accepts
  `attrs=` and honours its nonce — so our `Media` plugs into that tag on 6.2,
  while `with_nonce()`/constructor cover older Django.

## Docs

- `README.rst` — the single doc: assets, import-map merging via `js_asset.Media`,
  rendering in views/admin, and CSP nonces across Django 4.2 → main.

## Lint

`prek` / `ruff` (config in `pyproject.toml`).

## Commit Style

Commit without the `Co-Authored-By` attribution line (no `--co-author` / no `Co-Authored-By: Claude` trailer).
