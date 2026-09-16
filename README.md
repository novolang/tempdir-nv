# tempdir-nv

A **temporary directory** is a directory a program creates to work in
and deletes when it is done. A test that writes files needs one; so does
a program that builds an output before moving it into place. The API and
the safety rules here are
[tempfile](https://docs.rs/tempfile)'s in Rust and
[`tempfile.TemporaryDirectory`](https://docs.python.org/3/library/tempfile.html)'s
in Python. This package brings them to novo-lang, where they have to be
spelled differently: the language has no destructors, so nothing runs at
the end of a scope and no value can be consumed to stop something
running.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

A **root** is a directory this package created, or one a caller named on
purpose. It is what licenses a removal: nothing is deleted that is not
under the root the value was made in. `TmpRoot` is that path, and it
travels with every temporary directory and file.

A **naming policy** says what a generated name looks like: a prefix, a
number of random characters, a suffix, and how many times to retry when
the name is taken. `TmpNaming` is that policy, and the same value is
reused for every fixture in a program.

A **temporary directory** is a `TmpDir`: its root, its path, and its
**disposition** — whether closing it removes the tree or leaves it. A
**temporary file** is a `TmpFile`, the same shape for one file.

Closing is explicit and it is total. `tmpdir.close` answers a **report**
saying how many files, directories and links it removed and which paths
would not go, with the reason for each. It raises nothing: a teardown
that raised would turn a passing test red for a reason that is not the
test's, and inside a failing test it would replace the real failure with
its own.

Removing a tree is planned before it is performed. `tmpclean.plan_removal`
walks the tree and answers a list of entries, deepest first, each with
its kind and its depth. `tmpclean.remove_planned` performs exactly that
list. A caller can print the plan, assert on it, or refuse it. The walk
reads and the removal writes, and the effect rows say so.

`std.fs` has no recursive delete — `fs.delete` removes a file or an
empty directory — so this package writes that walk, which makes it the
most dangerous code any test library ships. The root check and the plan
are what contain it.

## Install

```
novo pkg add tempdir-nv
```

## Example

```novo
use std.fs
use tmpname
use tmpdir

// The body is a named function. A lambda body is one expression, and a
// fixture body is several statements.
fn writes_and_reads_back(d: TmpDir) -> Bool [fs]
    // `child` refuses a name that is not one path component, so it
    // cannot reach outside the directory.
    match tmpdir.child(d, "note.txt")
        Err(_) => false
        Ok(p)  =>
            let _ = fs.write(p, "hello")
            fs.read(p) == Some("hello")

fn main() [io, fs, rand]
    // The directory is created, the function runs, and the tree is
    // removed whether the function returned or panicked.
    match tmpdir.with_temp_dir(tmpname.prefixed("novo-"), writes_and_reads_back)
        Ok(ok) => println("${ok}")
        Err(e) => println(e.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: tempdir-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `tmproot` | The root: the value itself, the four ways to obtain one, the path arithmetic that decides what is under it, and the one error enum the whole package answers with. |
| `tmpname` | The naming policy: the prefix, token length, suffix and retry count, the candidate name for one token, the single function that draws entropy, the alphabet, and the two predicates a sweeper needs. |
| `tmpdir` | The directory: its disposition, five constructors, the two ways to build a path inside it, persisting, closing, and four shapes that create and close around one function. |
| `tmpfile` | The file: the creation flags as a value, four constructors, reading and writing it, persisting, closing, two scoped shapes, and the write-and-rename idiom written once. |
| `tmpclean` | The removal: the kinds of entry, the plan, the report and its readers, the three ways to perform a removal, and the first half of a sweeper for leaked fixtures. |

## How to choose an entry point

**`tmpdir.with_temp_dir` is the shape that cannot be forgotten.** It
creates, runs a function, and closes whatever happened.
`tmpdir.with_temp_dir_io` is the same for a body that also prints or
runs a subprocess, and `tmpdir.with_temp_dir_in` is the same under a
root the caller already has.

**`tmpdir.with_temp_dir_kept_on_failure` keeps the tree when the body
answers false.** Clean up after a pass, leave the evidence after a
failure, and report where it was left.

**`tmpdir.create_dir` and `tmpdir.close` are the primitive.** Take them
when the fixture outlives the function that made it: a harness with
separate setup and teardown hooks, or a directory shared by a whole
file of tests.

**`tmproot.system_root` finds the platform temporary directory and
`tmproot.root_at` takes one the caller names.** `tmproot.root_make`
creates it. Pass a root explicitly and the constructors need no
environment at all.

**`tmpfile.replace_atomically` is the write-and-rename idiom.** It
writes the content to a temporary file beside the destination and
renames it over, so a reader of the destination never sees a half-written
file.

**`tmpclean.plan_removal` then `.remove_planned` when the plan matters,
and `tmpclean.remove_tree` when it does not.**

## The rules a user needs

1. **Nothing is removed that is not under its root.** Every `TmpDir` and
   `TmpFile` carries the `TmpRoot` it was made in, and every removal
   checks. A value built by hand, or one whose path a caller rewrote, is
   refused with `TmpOutsideRoot` naming both the root and the path.
2. **The root check is textual and does not resolve symbolic links.**
   `tmproot.contains` normalises both paths and compares. On macOS
   `/tmp` is itself a symbolic link, so a caller comparing a resolved
   path against an unresolved root will see them disagree. Link
   behaviour lives in the walk, which has the directory open anyway.
3. **`persist` answers a new value; it does not cancel anything.** There
   is no destructor to stop, so `tmpdir.persist(d)` returns a `TmpDir`
   whose disposition is `TmpKeepOnClose` and the caller keeps it. A
   `persist(d)` written as a statement and discarded does nothing at
   all.
4. **`close` is total and answers a report.** `TmpRemoveReport` carries
   the counts of files, directories and links removed and every path
   that would not go with its reason. A directory whose disposition is
   `TmpKeepOnClose` answers a report of zeroes, which is the same answer
   as "it was already gone", and both are true answers to what the
   teardown removed. `tmpdir.close_ok` is the one-line form.
5. **Closing twice is not an error a caller has to prevent.** The second
   close reports `TmpAlreadyClosed` rather than silently deleting
   whatever now lives at that path.
6. **A removal does not stop at the first failure.** A teardown that
   abandoned the rest of the tree because one file was busy would leak
   far more than it cleaned. `tmpclean.first_failure` is there for a
   caller that wants one reason instead of the list.
7. **A symbolic link is removed, never entered.** `TmpEntrySymlink` is
   its own kind in a plan, so a reader can see it, and the walk does not
   descend into one. A link inside a fixture pointing at somebody's home
   directory is why this is in the type rather than in a comment.
8. **A plan is deepest first, with the top directory last.** That is the
   only order in which `fs.delete` can succeed, and it is the order
   `remove_planned` performs. `TmpEntry.depth` counts from the root at
   0, and `tmpclean.plan_complete` says whether a depth-limited plan
   reached the bottom.
9. **Planning an absent path is not an error.** The plan is empty,
   because a teardown that runs twice should be quiet the second time.
10. **There is no function that answers a unique name.** Whatever such a
    function returned was true when it returned and is a race by the
    time the caller creates it, which is why `mktemp(3)` and `tmpnam`
    were deprecated. What is published instead is
    `tmpname.candidate(policy, token)`, which is pure, and constructors
    that create exclusively and draw another token on a collision, up to
    `TmpNaming.attempts` times before `TmpNameExhausted`.
11. **`tmpname.draw_token` is the only function that draws entropy.**
    `tmpname.encode_token` declares no effects, so a test that pins the
    naming scheme needs no randomness, and a test that wants a collision
    hands the same integer twice. Everything that creates a named file
    inherits `[rand]` from `draw_token`.
12. **The alphabet is `23456789abcdefghijkmnpqrstuvwxyz`.** No `0`, `1`,
    `l` or `o`, because temporary paths are read aloud in bug reports;
    no upper case, because macOS and Windows filesystems are
    case-insensitive by default and a name that cannot collide with its
    own case-folding cannot collide with itself. The default policy is
    no prefix, no suffix, six token characters and ten attempts, and
    `tmpname.name_space` answers how many names a policy can produce.
13. **A name that is not one path component is refused before it reaches
    the filesystem.** `tmpname.component_ok` is false for the empty
    string, for `.` and `..`, for anything with a path separator and for
    anything with a NUL. It is the check `tmproot.child_path` runs, so
    `tmpdir.child(d, "../../etc")` answers `TmpBadName`.
14. **Temporary files are created exclusively, at mode `0o600`.**
    `tmpfile.exclusive_create` is that policy as a value, so a test can
    assert it. Owner read and write and nothing else, because a
    temporary file in a world-readable directory is the other half of
    the classic defect.
15. **The environment is read only to find the platform temporary
    directory.** `tmproot.system_root` and `.system_root_or` read
    `TMPDIR` and declare `[io]`; the seven constructors and scoped
    shapes that call one of them declare it too. Every constructor that
    takes a `TmpRoot` declares no `[io]` at all, and
    `tmpclean.remove_tree`, `tmpdir.close` and `tmpfile.close` are
    `[fs]` alone, so tearing a fixture down needs neither the
    environment nor entropy. `[io]` rather than `[fs]` for an
    environment read is SPEC section 5.1: that label covers the
    environment as well as pipes and processes.
16. **The scoped shapes come in two effect rows, and that is a language
    limit.** A function-typed parameter cannot bind an effect parameter:
    SPEC section 5.6 gives effect parameters to traits, and a free
    function binds one only through a trait bound. So the body of
    `with_temp_dir` is `[fs]` and the body of `with_temp_dir_io` is
    `[fs, io]`, and a body needing anything else — a network call, for
    instance — uses `create_dir` and `close` directly.
17. **`tmpname.looks_generated` is a necessary condition, not proof of
    ownership.** Two programs with the same prefix produce names of the
    same shape, which is why `tmpclean.leaked_under` still takes a root.

## What would have to change elsewhere

`std.fs` has no exclusive create today. `fs.open_write` creates a file
if it is absent and opens it if it is not, with no `O_EXCL` and no mode
argument, and the only atomic creations are `fs.temp_file()` and
`fs.temp_dir()`, which take neither a prefix nor a root and always use
the platform temporary directory.

So `tmpfile.create_file` and `tmpdir.create_dir` are implementable over
the standard library as it stands, by delegating to those two. Every
constructor that takes a `TmpRoot` or a naming policy needs an
`fs.open_exclusive(path: Str, mode: Int) -> ?Int` and an
`fs.mkdir_exclusive(path: Str, mode: Int) -> Bool`, which do not exist
yet. The interface is published with the signatures it should have: a
temporary-file package whose named constructors were quietly
non-atomic would be worse than one that does not build yet.

## What is not included

- **Automatic cleanup at the end of a scope.** The language has no
  destructors. `close` and the scoped shapes are what replace them.
- **A recursive delete in the standard library.** This package writes
  the walk; see rule 7 and rule 8.
- **A function that answers a unique name.** See rule 10.
- **Resolving symbolic links in the root check.** See rule 2.
- **Dependencies.** Everything here is `std.fs`, `std.env` and
  `std.rand` plus path arithmetic over `Str`. A package whose whole
  purpose is to be safe to depend on in a test should not bring
  anything else with it.
- **Running on a microcontroller.** No such claim is made. The package
  exists to touch a filesystem.

## Related packages

- [snapshot-nv](https://novo-lang.org/packages/snapshot-nv) stores test
  output in files and is the kind of package whose own tests need a
  directory of their own.
- [httpmock-nv](https://novo-lang.org/packages/httpmock-nv) is the other
  test fixture that must be torn down by hand, and it publishes the same
  two shapes — a primitive pair and a scoped form — for the same reason.
- `std.fs` in the standard library is everything underneath: creating,
  listing, deleting and renaming. `std.env` supplies the environment
  read and `std.rand` the name token.

## Tests

```bash
novo test --isolate tests/tmpfile_tests.nv   # 14 tests: the file, the flags and the atomic replace
novo test --isolate tests/tmpdir_tests.nv    # 13 tests: the disposition, close, and the scoped shapes
novo test --isolate tests/tmpclean_tests.nv  # 12 tests: the plan, the walk and the report
novo test --isolate tests/tmproot_tests.nv   # 11 tests: the root and the path arithmetic
novo test --isolate tests/tmpname_tests.nv   #  9 tests: the policy, the token and the alphabet
```

The behaviour asserted is tempfile's, and the argument for refusing a
unique-name function is `mktemp(3)`'s own deprecation notice.

The path arithmetic, the naming and the reports declare no effects, so
much of the suite runs with no disk at all. `tmproot_tests.nv` asserts
that a sibling sharing a prefix is not contained, that a relative path
never is, and that normalisation is what `contains` compares.
`tmpname_tests.nv` asserts that the same token gives the same candidate,
that the alphabet leaves out the characters people misread, and that
`component_ok` is the check `child_path` runs. `tmpclean_tests.nv`
asserts that a plan is deepest first and ends at the directory itself,
that a path outside the root is refused before anything runs, and that
`remove_planned` rechecks the root rather than trusting the plan it was
handed.

The tests compile today and fail at run, each on the
`not implemented: tempdir-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `tmproot.system_root`, `.system_root_or`, `.root_at`, `.root_make`, `.root_unchecked` | no |
| `tmproot.root_path`, `.join`, `.child_path`, `.contains`, `.normalise` | no |
| `tmproot.is_absolute`, `.base_name`, `.parent_name` | no |
| `tmproot.error_text`, `.error_path`, `.is_caller_fault`, `tmproot.TmpError.message` | no |
| `tmpname.naming`, `.prefixed`, `.with_prefix`, `.with_suffix`, `.with_token_len`, `.with_attempts` | no |
| `tmpname.candidate`, `.draw_token`, `.encode_token`, `.alphabet`, `.name_space` | no |
| `tmpname.component_ok`, `.looks_generated` | no |
| `tmpdir.create_dir`, `.create_dir_in`, `.create_sub_dir`, `.make_child_dir`, `.make_child_path` | no |
| `tmpdir.dir_path`, `.child`, `.child_at`, `.exists`, `.keeps` | no |
| `tmpdir.persist`, `.persist_to`, `.close`, `.close_ok` | no |
| `tmpdir.with_temp_dir`, `.with_temp_dir_io`, `.with_temp_dir_in`, `.with_temp_dir_kept_on_failure` | no |
| `tmpfile.exclusive_create`, `.with_mode`, `.open_flags` | no |
| `tmpfile.create_file`, `.create_file_in`, `.create_file_under`, `.make_child_file` | no |
| `tmpfile.file_path`, `.write_str`, `.write_bytes`, `.read_str`, `.open_read`, `.open_write` | no |
| `tmpfile.persist`, `.persist_as`, `.close`, `.exists`, `.keeps` | no |
| `tmpfile.with_temp_file`, `.with_temp_file_io`, `.replace_atomically` | no |
| `tmpclean.plan_removal`, `.plan_removal_to_depth`, `.plan_complete`, `.default_depth_limit` | no |
| `tmpclean.remove_planned`, `.remove_tree`, `.empty_tree` | no |
| `tmpclean.report_clean`, `.report_total`, `.report_line`, `.first_failure`, `.render_plan` | no |
| `tmpclean.leaked_under` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
