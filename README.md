# tempdir-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Temporary files and directories for novo-lang, and the answer to the
question a port of [tempfile](https://docs.rs/tempfile) has to answer
first: **novo-lang has no destructors**, so nothing runs at the end of a
scope and nothing consumes a value to stop it running. Copying
tempfile's API without `Drop` and without move semantics produces
something that looks safe and leaks.

- `tmproot` — `TmpRoot`, the directory this package created and may
  therefore delete, plus the path arithmetic that decides what is under
  it;
- `tmpname` — the naming policy, the token encoding, and why there is
  no `unique_name` function;
- `tmpdir` — `TmpDir`, `close`, `persist`, and `with_temp_dir`;
- `tmpfile` — `TmpFile`, the exclusive-create flags, and the
  write-and-rename idiom;
- `tmpclean` — the recursive removal: planned as a value, then
  performed.

```
novo pkg add tempdir-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use tmpname
use tmpdir
use std.fs

// The body is a NAMED function.  A lambda body is one expression, a
// fixture body is several statements, and at `@tier(embedded)` a
// lambda that becomes a value is refused.
fn writes_and_reads_back(d: TmpDir) -> Bool [fs]
    match tmpdir.child(d, "note.txt")
        Err(_) => false
        Ok(p)  =>
            let _ = fs.write(p, "hello")
            fs.read(p) == Some("hello")

fn main() [io, fs, rand]
    match tmpdir.with_temp_dir(tmpname.prefixed("novo-"), writes_and_reads_back)
        Ok(ok) => println("${ok}")
        Err(e) => println(e.message())
```

The directory is created, the function runs, the tree is removed — and
the removal happens whether the function returned or panicked.

## The load-bearing interface: `TmpDir` carries its own disposition

```novo ignore
pub enum TmpDisposition
    TmpRemoveOnClose
    TmpKeepOnClose

pub struct TmpDir
    root: tmproot.TmpRoot
    path: Str
    disposition: TmpDisposition
```

Three things follow from putting the decision in the value, and they
are why the design holds.

**`persist` is an ordinary function.** In Rust, `into_path` works
because it consumes the `TempDir` so the destructor cannot run
afterwards. Here there is no destructor to stop, so `persist(d)` answers
a NEW `TmpDir` whose disposition is `TmpKeepOnClose` and the caller
keeps it. That is the honest spelling: a statement-shaped `persist(d)`
that discarded its result would have done nothing at all, and this
version at least reads like what it is.

**`close` is total.** It answers a `TmpRemoveReport` — files, directories
and links removed, and every path that would not go, with the reason. It
raises nothing. A teardown that raises turns a passing test red for a
reason that is not the test's, and a teardown that raises inside a
failing test replaces the real failure with its own. A `TmpKeepOnClose`
directory answers an all-zero report, which is the same answer as "it
was already gone", and both are true answers to "what did the teardown
remove".

**The root travels with the value.** `close` removes nothing that is
not under the `TmpRoot` the directory was made in, and `TmpRoot` is a
directory this package created or that a caller named on purpose. So a
`TmpDir` built by hand, or one whose `path` a caller rewrote, is
harmless: it is refused with `TmpOutsideRoot` naming both paths.

### And both shapes are published, because neither is enough

`create_dir` + `close` is the primitive. It composes, it survives an
early return, and it is what a harness with separate setup and teardown
hooks needs — the fixture outlives the function that made it.

`with_temp_dir(naming, named_fn)` is the shape that cannot be forgotten.

A package that shipped only the callback could not express a fixture
shared by a whole suite; one that shipped only the pair could not stop a
person forgetting the second half. So there are both, and
`with_temp_dir_kept_on_failure` is the third shape a real test suite
reaches for: clean up after a pass, leave the evidence after a failure.

### Why the callback family is finite

A function-typed **parameter** cannot bind an effect parameter. SPEC
§ 5.6 gives effect parameters to traits, and a free function binds one
only through a trait bound — `fn f<S: Read[e]>(…) [e]`. There is no way
to write `fn(TmpDir) -> T [e]`.

So the rows are concrete and there are two of them: `[fs]` for a body
that only uses the directory, and `[fs, io]` for one that also prints or
shells out. A body that needs `[net]` uses `create_dir` and `close`
directly. The alternative — a one-method trait with an effect parameter,
implemented per test body — costs a struct and an `impl` for every
fixture and buys back rows nobody has asked for.

## The removal is planned before it is performed

`std.fs` has no recursive delete: `fs.delete` removes a file or an
**empty** directory. So this package writes the walk, and that walk is
the most dangerous function any test library ships. Two decisions
contain it.

**The plan is a value.** `plan_removal` answers `[TmpEntry]` — deepest
first, each with its kind and its depth — and `remove_planned` performs
exactly that list. A caller can print it, assert on it, or refuse it.
Planning reads and performing writes, and the effect rows say so.

**A link is removed, never entered.** `TmpEntrySymlink` is its own kind,
the walk does not descend into one, and the removal unlinks the link
itself. A symbolic link inside a fixture pointing at somebody's home
directory is the reason this is stated in the type rather than in a
comment.

A removal does not stop at the first failure, either: a teardown that
abandoned the rest of the tree on one busy file would leak far more than
it cleaned.

## The naming, and the function that is not here

There is no `unique_name(root) -> Str`. A function with that signature
cannot be used correctly — whatever it answers was true when it answered
and is a race by the time the caller creates it, which is why `mktemp(3)`
and `tmpnam` were deprecated.

What is published instead is a **candidate** and an **attempt count**:
`tmpname.candidate(policy, token)` is pure, the constructors try to
create it exclusively, and a collision draws another token up to
`attempts` times before `TmpNameExhausted`. `encode_token` is `[]` and
`draw_token` is the package's only `[rand]` function, so a test that
pins the naming scheme needs no entropy and a test that wants a
collision hands the same integer twice.

The alphabet is `23456789abcdefghijkmnpqrstuvwxyz`: no `0`, `1`, `l` or
`o`, because temp paths are read aloud in bug reports; no upper case,
because macOS and Windows filesystems are case-insensitive by default
and a name that cannot collide with its own case-folding cannot collide
with itself.

## The missing standard-library row

**`std.fs` has no exclusive create today.** `fs.open_write` creates a
file if it is absent and opens it if it is not — no `O_EXCL`, no mode
argument — and the only atomic creations are `fs.temp_file()` and
`fs.temp_dir()`, which take neither a prefix nor a root and always use
the platform temp directory.

So of the constructors published here, `tmpfile.create_file` and
`tmpdir.create_dir` are implementable over the standard library as it
stands, by delegating; every constructor that takes a `TmpRoot` or a
naming policy needs an `fs.open_exclusive(path: Str, mode: Int) -> ?Int`
and an `fs.mkdir_exclusive(path: Str, mode: Int) -> Bool` that do not
exist yet.

The interface is published with the signatures it should have, and this
is written down rather than worked around: a temporary-file package
whose named constructors were quietly non-atomic would be worse than one
that does not build yet.

## The layer

`host`, and the rows are narrow on purpose.

| what | row |
| --- | --- |
| creating, listing, removing, renaming | `[fs]` |
| reading `TMPDIR` — four functions, all in `tmproot` | `[fs, io]` |
| drawing a name token — one function in `tmpname` | `[rand]` |
| path arithmetic, naming, planning, reports | `[]` |

`tmpclean.remove_tree`, `tmpdir.close` and `tmpfile.close` are `[fs]`
alone, so tearing a fixture down needs neither entropy nor the
environment — which matters for a test running in a sandbox that grants
one and not the other.

`[io]` rather than `[fs]` for the environment read is not a slip:
SPEC § 5.1's `[io]` covers the environment as well as pipes and
processes, and `env.get` declares it.

## No dependencies

Everything here is `std.fs`, `std.env` and `std.rand` plus path
arithmetic over `Str`. A package whose whole purpose is to be safe to
depend on in a test should not drag a closure in with it.

## Reference implementation

[tempfile](https://docs.rs/tempfile) (Rust), with
[`tempfile.TemporaryDirectory`](https://docs.python.org/3/library/tempfile.html)
(Python) for the context-manager shape that `with_temp_dir` is the
novo-lang spelling of. The safety argument is
[`mktemp(3)`](https://man7.org/linux/man-pages/man3/mktemp.3.html)'s own
deprecation notice.

## Status

Interface only. Every body is `todo()`; `novo pkg build` type-checks and
effect-checks the whole surface, and `novo test tests` is red until the
bodies land.
