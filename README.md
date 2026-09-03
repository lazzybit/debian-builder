# debian-builder

Automated Debian package builder powered by GitHub Actions. Each package is a
self-contained directory with its own version tracker, update checker, and
build script.

## Directory structure

```
<package>/
├── VERSION          # Current upstream version (plain text, no trailing newline)
├── check-update     # Script: checks if a newer upstream release exists
├── build            # Script: builds the .deb
├── templates/       # dpkg control templates (control, postinst, prerm, etc.)
└── out.deb          # Build output (gitignored)
```

## Script conventions

Both scripts are invoked from the repository root with the script path (e.g.
`bash anki/build`). Each script detects its own directory and reads resources
relative to it.

### `check-update`

| Condition              | Exit code | stdout           | stderr              |
| ---------------------- | --------- | ---------------- | -------------------- |
| Already up to date     | 0         | (ignored)        | (ignored)           |
| New version available  | 1         | `<version>`      | (ignored)           |
| Runtime error          | 2+        | (ignored)        | Human-readable error |

When a new version is available, stdout must contain **only** the version
string (e.g. `26.06`). The workflow uses this to update `VERSION`.

### `build`

| Condition    | Exit code | stdout      | stderr                 |
| ------------ | --------- | ----------- | ---------------------- |
| Success      | 0         | `out.deb`   | Progress / diagnostics |
| Build error  | 1         | (ignored)   | Human-readable error   |
| Prerequisite error | 2+  | (ignored)   | Human-readable error   |

The output file **must** be `./out.deb` relative to the package directory.

## GitHub Actions

### `check-updates.yml`

- **Triggers:** daily schedule (midnight UTC) + manual dispatch
- Runs `check-update` for every package directory, updates `VERSION` if a
  new release is found, and commits each update separately before pushing.

### `build-deb.yml`

- **Triggers:** `*/VERSION` changes on `main`, successful completion of
  `check-updates.yml`, or manual dispatch
- Detects which packages have a changed `VERSION`, runs `build` for each,
  and updates the `latest` GitHub Release with unversioned package assets.

## Adding a new package

1. Create a directory under the repo root (e.g. `obsidian/`).
2. Add `VERSION` with the current upstream version.
3. Write a `check-update` script following the convention above.
4. Write a `build` script following the convention above.
5. Create `templates/control`, `templates/postinst`, `templates/prerm` as
   needed by `dpkg-deb`.
6. Commit and push — the workflows will pick it up automatically.
