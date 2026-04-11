# brian_bedrocks

## Python
1. When using Python, use version 3.13.
2. For Python Virtual environments, prefer uv over other options.
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
