# ci-workflows conventions

- Reusable workflows must live directly in `.github/workflows/` (GitHub doesn't support subfolders there), so repo types are file-name prefixes: `common-`, `go-`, `image-`, `coredns-plugin-`.
- Types are composable building blocks (an image repo uses `go-pr` + `image-pr` + `image-release`); `image-` stays language-agnostic.
- A `common-` file only exists when two or more types use it.
- Consumer repos only reference type files and `renovate/<type>.json`, never `common-*`.
- Type files call common ones with a local `./.github/workflows/common-*.yml` reference (same commit, no version drift).
- Triggers, permissions and the `ci-ok` required-check job stay in the consumer repo.
- Concurrency and permissions inside reusable workflows are set at job level.
- No releases or tags: consumers reference `@main`; this repo only maintains `main` and Renovate keeps its dependencies up to date.
