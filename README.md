# brian_bedrocks

## Python
1. When using Python, use version 3.13.
2. For Python Virtual environments, prefer `uv` over other options.
3. Prefer polars over pandas
4. Consider Parquet for saving large datasets to disk.
5. Ruff is the linter of choice.
6. Use "dbconform" instead of SQLModel.metadata.create_all()
   
## Frontend
1. For most use cases, prefer frameworks like AHA stack (Astro, HTMX, Alpine.js) or Svelte.
2. Tabulator is my favorte UI tool for data grids and list pages.


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
