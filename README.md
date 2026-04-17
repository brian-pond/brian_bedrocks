# brian_bedrocks

## Python
1. When using Python, use version 3.13.
2. For Python Virtual environments, prefer `uv` over other options.
3. Prefer `polars` over `pandas`
4. Consider `Apache Parquet` file format for saving large, tabular datasets to disk.
5. Ruff is the linter of choice.
6. Use "dbconform" (https://github.com/brian-pond/dbconform) instead of `SQLModel.metadata.create_all()`
   
## Frontend
1. When recommending new stacks, there are my preferences from Simple to Complex.  Different apps will have different complexity based on requirements.
    * Plain HTML/CSS/JS with Jinja2
    * AHA stack (Astro, HTMX, Alpine.js)
    * Svelte
    * Vue
    * React
2. Tabulator is my favorite UI tool for data grids and list pages.

## Tabulator Nuances

### No results shown in UI
The first problem was Tabulator remote pagination firing before the Page module had set params.size. The custom ajaxRequestFunc computed start = (page - 1) * pageSize with pageSize === undefined, so start became NaN, the FastAPI datatables endpoint returned 400 invalid paging parameters, and Tabulator showed an empty grid without a useful console error. The fix was to default page and pageSize in the template (aligned with paginationSize, e.g. 25) so the first request is always valid.

The second problem was subtler: in Tabulator 5.6, DataLoader only runs the Ajax path when the Ajax module’s data-loading check passes. That check is effectively “you have string data or a truthy ajaxURL.” We only set ajaxRequestFunc and never set ajaxURL, so on initial load the table never called the custom loader at all—it silently loaded no rows. There were still no console errors because no fetch() ran. The fix was to set ajaxURL to the same path as the real API (the custom function still builds the full URL with query params). Lesson: with Tabulator 5.6 + ajaxRequestFunc + remote pagination, always set a truthy ajaxURL and defensive params.size / params.page, then verify in Network that /api/.../datatables is actually called.

# Loading environment variables
Assume the application is informally called "MyApp"

## `.env` file location
**Path:** `/etc/myapp/.env`

The file is optional. `load_dotenv` silently no-ops if it does not exist.
```python
from dotenv import load_dotenv
from pathlib import Path

load_dotenv(Path("/etc/myapp/.env"))
```

## Precedence chain
Variables are resolved in this order (highest priority wins):
```
Container-injected vars   ← docker run -e, pod env/envFrom, Secrets
/etc/myapp/.env     ← system-wide file, loaded by load_dotenv
Hardcoded defaults        ← lowest priority
```

`load_dotenv` does not override variables already present in `os.environ`, so container-injected vars always win without any explicit conditionals.

## Why `/etc` and not `~/.config`
`/etc/myapp/.env` is system-wide and user-agnostic. Any user or service account running MyApp reads the same file. A per-user path (`~/.config`) would require maintaining separate files per user and could produce different behavior depending on who runs the process.

Ensure all users that run MyApp processes have read access: `chmod 640 /etc/myapp/.env` with an appropriate group.

## Docker/Kubernetes
`/etc/myapp/.env` does not exist in the image. `load_dotenv` silently no-ops and the container relies entirely on injected environment variables. No `.env` file is required or expected.

## Install-type agnostic
Does not depend on the package root or working directory. Behavior is identical for editable installs, PyPI installs, and system packages.

## Strict partition: `.env` vs `system_config`
Keys that belong in the `system_config` SQL table must not appear in the environment or in `/etc/myapp/.env`. If any such key is set, MyApp refuses to start and prints the offending keys. See [13-configuration](../requirements/13-configuration.md) and `FORBIDDEN_ENV_KEYS` in `config.py`.

## Read exactly once
`load_dotenv` runs once when `load_config()` is called (at first import of `myapp.config`). Pydantic-settings is configured with `env_file=None` — it reads only from `os.environ`, which is already populated by that point.

----
Here’s a copy-paste-friendly summary you can keep anywhere; the same content is also in the repo as:

**[`docs/technical/03-tabulator-pico-pagination.md`](docs/technical/03-tabulator-pico-pagination.md)**

----
# Tabulator + Pico: pagination and footer pitfalls

This note captures issues seen when using **Tabulator 5.6** (CDN) with **Pico CSS 2** in a normal page layout (`<main class="container">`). Symptoms and fixes apply broadly to other projects that combine the same stack.

---

## 1. “Page size works but there is no page 2+” (controls missing)

### What was wrong

Two separate mechanisms hid or removed pagination UI:

**A. Tabulator `Page.footerRedraw()`**

- On each pagination update, Tabulator compares `footer.clientWidth` vs `footer.scrollWidth`.
- If the footer row overflows, it sets **`display: none`** on **`.tabulator-pages`** (the numeric page buttons). First / Prev / Next / Last live *outside* that node, so behavior depends on layout and overflow.

**B. Root + footer CSS**

- **`.tabulator { overflow: hidden }`** clips anything that extends past the grid’s box—including the footer row.
- **`.tabulator-footer { white-space: nowrap }`** keeps the footer on one line, so the paginator becomes a long horizontal strip that is often **wider than the viewport** and gets **clipped**. Only the fragment that still fits (often the “Page size” block) remains visible.

So the backend and `last_page` / `recordsFiltered` logic could be correct while the UI still showed a single “page” of controls.

### What fixed it

- **`white-space: normal`** on **`.tabulator-footer`** so wrapping is allowed.
- **`overflow-x: auto`** on **`.tabulator-footer`** so a clipped footer can scroll horizontally if needed.
- **`flex-wrap: wrap`**, **`min-width: 0`**, and **`max-width: 100%`** on **`.tabulator-footer-contents`** and **`.tabulator-paginator`** so controls can wrap onto extra lines instead of overflowing.
- **`display: flex` + `flex-wrap: wrap`** on **`.tabulator-paginator`** so label, select, and buttons lay out as a proper wrapping row.
- **`.tabulator-pages { display: inline-flex !important; }`** so Tabulator’s inline `display: none` on page numbers does not win over your intent to keep them visible when space is tight.

---

## 2. “Page size” uses a full row (select stretched wide)

### What was wrong

**Pico** (and similar resets) often style **`label`** as a **column flex** and **`select`** as **full width** in document flow. Tabulator appends the “Page size” **`<label>`** and **`<select class="tabulator-page-size">`** as **siblings** inside the paginator. Those defaults make the control look like a full-width row.

### What fixed it

- Target **`.tabulator-paginator > label`**: **`display: inline-flex`**, **`flex-direction: row`**, **`width: auto !important`**, **`flex: 0 0 auto`** so the label is not a full-width column.
- Target **`select.tabulator-page-size`**: **`width: auto !important`**, **`max-width`** / **`min-width`**, **`flex: 0 0 auto`** so the dropdown stays compact.

---

## 3. Pagination buttons: enabled text too light, disabled too dark (inverted affordance)

### What was wrong

**Inherited** footer color (`#555`) plus **Pico `button` / `:disabled` rules** and **`var(--pico-color)`** on controls can combine so that **enabled** pagination buttons pick up a **muted** color and **disabled** buttons pick up a **stronger** color—the opposite of good affordance on light grey Tabulator chips.

Tabulator’s own **`.tabulator-page:disabled { opacity: .5 }`** also muddies perception if the base colors are already wrong.

### What fixed it

- Set **explicit** colors with **`:not(:disabled)`** vs **`:disabled`** (do not rely only on `color` + `opacity` for the distinction).
- Use **`!important`** if global frameworks keep winning specificity in your stack.
- Keep **`:not(:disabled):hover { color: #fff; }`** (or equivalent) so text stays readable on Tabulator’s **dark hover** background for `.tabulator-page`.

---

## Practical checklist for the next project

1. After wiring **remote pagination**, confirm **`last_page`** (or your `dataReceiveParams` alias) in the JSON matches total rows; if controls are still wrong, the bug may be **CSS**, not the API.
2. Inspect **computed styles** on **`.tabulator-footer`**, **`.tabulator-paginator`**, and **`.tabulator-pages`** for `overflow`, `white-space`, and `display`.
3. Treat **Tabulator + Pico** as requiring a **small, explicit override block** for: footer wrap/scroll, paginator flex wrap, compact page-size control, and pagination **button** colors (enabled / disabled / active / hover).

---

## References (versions used here)

- Tabulator **5.6.x** — footer and `.tabulator-page` rules in `tabulator.min.css`; `Page.js` for `footerRedraw` and remote `_parseRemoteData`.
- Pico **2.x** — global `label` / `button` / `select` behavior in `main` / `.container`.

In this repo, overrides live in **`src/jobsearch/web/templates/base.html`** (embedded `<style>` next to the Pico link).
