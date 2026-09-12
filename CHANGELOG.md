# Changelog

All notable changes to tempdir-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `tmproot` — `TmpRoot`, the directory this package created and may
  therefore delete; `system_root` (the one environment read),
  `root_at`, `root_make`, `root_unchecked`; `contains`, `normalise`,
  `child_path` and the path arithmetic they are built from; `TmpError`
  with a path in every variant, and `impl Error for TmpError`.
- `tmpname` — `TmpNaming` as a value, `candidate` pure over one token,
  `draw_token` as the only `[rand]` function, `encode_token`,
  `alphabet`, `name_space`, `component_ok` and `looks_generated`.
- `tmpdir` — `TmpDisposition` and `TmpDir`; `create_dir`,
  `create_dir_in`, `create_sub_dir`, `make_child_dir`,
  `make_child_path`; `child`, `child_at`; `persist`, `persist_to`,
  `close`, `close_ok`, `exists`; and the three callback shapes
  `with_temp_dir`, `with_temp_dir_io`, `with_temp_dir_in` plus
  `with_temp_dir_kept_on_failure`.
- `tmpfile` — `TmpOpen` and the exclusive-create flags as a value;
  `TmpFile`; the four constructors; `write_str`, `write_bytes`,
  `read_str`, `open_read`, `open_write`; `persist`, `persist_as`,
  `close`, `exists`; `with_temp_file` and `with_temp_file_io`; and
  `replace_atomically`, the write-and-rename idiom written once.
- `tmpclean` — `TmpEntryKind`, `TmpEntry`, `TmpRemoveReport`,
  `TmpFailure`; `plan_removal`, `plan_removal_to_depth`,
  `plan_complete`, `remove_planned`, `remove_tree`, `empty_tree`;
  the report readers; `render_plan`; and `leaked_under`, the first half
  of a sweeper.

### Known

- **`TmpDir` carries its own disposition, and `close` is total.** The
  language has no destructors, so `persist` cannot be "cancel the
  drop" — it answers a new value — and `close` answers a report rather
  than raising, because a teardown that raises replaces a real failure
  with its own.
- **Both shapes are published.** `create_dir` + `close` is the
  primitive, because a fixture can outlive one function;
  `with_temp_dir(naming, named_fn)` is the shape that cannot be
  forgotten. Neither is sufficient alone.
- **The callback family is finite and the reason is the language.** A
  function-typed parameter cannot bind an effect parameter (SPEC § 5.6
  gives them to traits), so the rows are concrete: `[fs]` and
  `[fs, io]`. A body needing anything else uses the primitive.
- **The removal is a plan before it is an action**, and a symbolic link
  is its own entry kind that the walk never enters.
- **There is no `unique_name`.** A name is a promise about the past;
  the package publishes a candidate and an attempt count instead.
- **The root check is textual and says so.** `contains` normalises and
  compares; it does not resolve symbolic links, and `/tmp` is a symlink
  on macOS. The link handling lives in the walk, which has the
  directory open anyway.
- **`std.fs` has no exclusive create**, so every constructor taking a
  root or a naming policy needs `fs.open_exclusive(path, mode)` and
  `fs.mkdir_exclusive(path, mode)`, which do not exist yet. Recorded
  here rather than worked around.
- **No dependencies**, and no device claim: this package is `[fs]` by
  its nature and has nothing to say at `@tier(embedded)`.
