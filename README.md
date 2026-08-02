# tbProjPacker — twinBASIC package

A twinBASIC package for packing and unpacking twinproj/twinpack files, with
no command-line dependency. Reference the package and call `ImpExp.Pack`,
`ImpExp.Unpack`, and `ImpExp.Test` directly from twinBASIC code.

This is the package counterpart to the [ImpExp console
app](https://github.com/GCuser99/tbImportExport). Same core logic; a
different, in-code surface. The two are maintained independently — this
package is a clean build, not a wrapper around the CLI.

## Public surface

Exactly three functions, all on the `ProjPacker` class, all returning `Boolean`
(`True` on success, `False` on failure):

```vba
Dim pp As New ProjPacker

' Pack a directory tree into a .twinproj / .twinpack file.
ok = pp.Pack(inputDir, outputFile, [overwrite], [logFile])

' Unpack a .twinproj / .twinpack file into a directory tree.
ok = pp.Unpack(inputFile, outputDir, [cleanFirst], [logFile])

' Run the self-test suite against a sample .twinproj / .twinpack.
ok = pp.Test(samplePath, [logFile])
```

### Arguments

| Function | Argument | Meaning |
|---|---|---|
| `Pack` | `inputDir` | Directory tree to pack. |
| | `outputFile` | Destination `.twinproj` / `.twinpack`. |
| | `overwrite` | Optional. `True` replaces an existing output file. Default `False` refuses to overwrite (see below). |
| | `logFile` | Optional. Path to a log file for this call's output. |
| `Unpack` | `inputFile` | `.twinproj` / `.twinpack` to unpack. |
| | `outputDir` | Destination directory. |
| | `cleanFirst` | Optional. `True` empties the target directory before unpacking. Default `False`. |
| | `logFile` | Optional. Path to a log file for this call's output. |
| `Test` | `samplePath` | Required. A real `.twinproj` / `.twinpack` to test against. |
| | `logFile` | Optional. Path to a log file for this call's output. |

## Error model

These functions **never raise**. Any failure is caught internally and
reported as a `False` return, with the reason emitted (see Output below).
A caller that only needs a yes/no checks the return value; a caller that
wants the reason reads the log or the immediate window.

```vba
Dim pp As New ProjPacker
If Not pp.Pack("C:\proj\MyApp", "C:\out\MyApp.twinproj") Then
    ' something went wrong -- reason was printed to Debug and, if a
    ' logFile was supplied, written there.
End If
```

The one failure surfaced the same way as any other is a log file that can't
be opened: if you pass a `logFile` and it can't be created, the operation
does not run and returns `False`. Asking for a log and silently getting none
would hide a problem, so it's treated as a failure like any other.

## Output

Every operation always calls `Debug.Print` for its progress and summary
lines. In the IDE these appear in the immediate window; in a compiled build
`Debug.Print` is skipped entirely, so it costs nothing. This is the intended
usage — the package is meant to be driven from the IDE.

Supplying `logFile` additionally writes those same lines to a file, as UTF-8
with Windows (CRLF) line endings, **overwriting** any existing file (each
call's log is that call's story). Because project output routinely contains
non-ASCII file names, the log is written as UTF-8 rather than the native
codepage.

## Guards (why a call may return False)

The package carries the same safety guards as the console tool:

**Export refuses to overwrite by default.** A `.twinproj` is the source of
truth for a project; unlike an unpacked tree, there is nothing to regenerate
it from once overwritten. So `Pack` returns `False` if the output file
exists, unless you pass `overwrite:=True`.

**Export refuses a tree with no `Settings`.** `Settings` holds the project
name, references, version, and compile options — content the IDE cannot
reconstruct if it is missing. Packing such a tree would produce a file that
opens with source intact but everything else blanked, so `Pack` refuses.
(`Settings` is normally a file, not a folder, so this rarely triggers; it
guards the case where it genuinely went missing, e.g. a version-control
round trip that dropped it.)

**Clean-first refuses a shallow target.** `Unpack` with `cleanFirst:=True`
empties the target before writing, but refuses a dangerously shallow path (a
drive root or a single top-level folder) rather than recursively delete it.

Unpacking is otherwise the safe direction — the tree it writes is a derived
artifact, and the `.twinproj` it came from is untouched — so `Unpack`
overwrites an existing tree freely, and `cleanFirst` is offered only as a
convenience for a tidy result.

## Files

| File | Scope | Contents |
|---|---|---|
| `ProjPacker.twin` | **Public class** | The only public surface: `Pack`, `Unpack`, `Test`. |
| `modOutput.twin` | Private module | `Say` sink: `Debug.Print` + optional UTF-8 log file. |
| `modImpExp.twin` | Private module | Parser, serializer, import, export. |
| `modShared.twin` | Private module | UTF-8, path/filesystem helpers, ordinal sort. |
| `modSelfTest.twin` | Private module | Test suite. |
| `modWin32.twin` | Private module | Win32 declares (toggle-able against WinDevLib). |
| `Entry.twin` | Private class | Tree node. |
| `ByteBuffer.twin` | Private class | Append-only output buffer. |
| `ByteReader.twin` | Private class | Bounds-checked little-endian cursor. |

Only `ProjPacker` is public. Everything else is private to the package, so it
won't collide with members of the referencing project.

## License

MIT © 2026 GCuser99
