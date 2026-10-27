# Changelog

## 27.10.2026
* **GEÄNDERT** (Build 20261027_1552): Use original commit text and write changelogs as UTF-8 with BOM

Drop the GitHub Models call, its fallback, and the models permission.
Keep changelog, version, and commit behavior, and prefix the generated
changelog files with a UTF-8 BOM so they match .editorconfig.

Co-authored-by: Dusa <Dusa@users.noreply.github.com>

Allow manual changelog runs and document the workflow

Add workflow_dispatch without changing push or workflow_call, grant
models: read for the GitHub Models call, pass the before SHA through
the environment, fix scrollIntoView in the changelog HTML, and add a
German README that matches the workflow.

Co-authored-by: Dusa <Dusa@users.noreply.github.com>

