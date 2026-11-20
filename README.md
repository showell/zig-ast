# zig-ast

**zig-ast reads Zig source the way Zig's own tokenizer and parser do, and
answers questions about it over a pipe: its tokens, its declarations, its
imports, and the exact edits to rename a name or move a declaration to
another file.**

The code here was written by AI coding agents (Claude), with human supervision.

It is built on `std.zig`, the tokenizer and parser the Zig compiler itself
uses, so it agrees with Zig about what the code says: a use is a use in the
code, outside comments and strings; `a.b` is told from `b`; and a name
shadowed in one scope keeps its own uses. That is what a lint, a
refactoring tool or a codemod needs, and what a regex only approximates.

It reads one file at a time, on its own, so a file of a half-finished
refactor works as well as one that compiles. The caller
writes every file: zig-ast answers with the edits, and the caller applies
them.

## Frames, and why

zig-ast is one long-running program, `zig-ast --serve`, that a tool starts
once and keeps for many questions, talking over stdin and stdout. Every
message each way is a **frame**: its length in bytes on a line of its own,
then exactly that many bytes. A request carries whole source files, and
those hold newlines, tabs and any other byte, so a length is the simplest
way for a reader to take them raw: it reads the count, then reads exactly
that many bytes, and the source arrives byte for byte as written. Inside a frame,
the fields are plain lines of tab-separated text, so any language can drive
it.

## An example

A two-import file, asked for its imports. The request is one frame: `101`,
its length, then the body. The body is the command, then `src` with the
source's length in bytes, then the source itself:

    101
    imports
    src	85
    const std = @import("std");
    const wire = @import("wire.zig");

    pub fn main() void {}

zig-ast's whole answer:

    zig-ast serve 1 0.17.1+patched.3
    52
    work	38
    i	import	std	std	1
    i	import	wire.zig	wire	2

The first line, written once when it starts, names the wire format (1) and
the Zig whose parser it carries. Then each answer is a frame the same way: a
length, then lines. `work 38` is what the request cost, for a caller that
budgets. Each `i` line is one import: its kind, the target, the name it is
bound to, and its line.

## The commands

- **`tokens`**: every token, as Zig's tokenizer sees it, in fixed-width
  binary records (tag, start, end, line), optionally only one tag.
- **`decls`**: each declaration, its kind, name, visibility, linkage and
  span, the comment above it, and the names it uses and reaches; optionally
  inside one container.
- **`imports`**: each `@import`, as above.
- **`rename`**: the edits that rename one or more names, in a line range or
  a container if asked, with the count of uses left in comments.
- **`move`**: the edits that move declarations from one file to another,
  qualifying the uses left behind, making a moved name `pub` where needed,
  and naming any import the destination now needs.

`main.zig`'s header gives each command's request lines and answer events in
full.

## The wire

- zig-ast first writes one line, `zig-ast serve <format> <zig version>`.
  The format is `wire.zig`'s `format`, 1 today. A change a reader must know
  about gets a new number, so a caller checks it before trusting an answer.
- Then each request is a frame, `<length>\n` and that many bytes, and each
  answer is a frame the same way.
- A frame's body is lines of fields joined by a tab, each field escaped
  (`\\`, `\t`, `\n`, `\r`).
- A request's first line is its command. A line `src<TAB><n>` is followed
  by n raw bytes, a source file, and a newline.
- An answer is events, read in order: `work <n>` charges n steps, a refusal
  is its own word (`parse_error <line> <tag>`, `invalid <line>`, and the
  others `main.zig`'s header lists), and then the values.
- A malformed request is answered `bad <why>`.

`wire.zig` is the format's one home.

## The pin: one Zig

zig-ast reads Zig with the parser of the Zig it is built with, so its pin
sets the syntax it reads. `zig-version` names it: a patched 0.17.1, from
[patched-zig-compiler](https://github.com/showell/patched-zig-compiler).
Code using syntax newer than that pin gets a `parse_error` naming its line.

## Building, testing and installing

```sh
zig build          # zig-out/bin/zig-ast, always ReleaseSafe
zig build test     # the format, then the tests
```

Its one outside dependency is
[zig-webbox](https://github.com/showell/zig-webbox) (its `src/zig_refs.zig`
reads a file's `@import`s), pinned by commit in `build.zig.zon`.

The tests hold the reader, the planner and the mover to fixtures and to a
corpus of real Zig, read whole: this repository's own files and zig-webbox's
`src/`. A rename there and back, or a move there and back, must give the
file it began with, and the two must agree on what is a use.

## How Shale uses it

[Shale](https://github.com/showell/shale) is a small scripting language for
working on code, and its Zig builtins (`zig_decls`, `zig_tokens`,
`zig_imports`, `zig_rename`, `zig_renames` and `zig_move`) stand on zig-ast.
Shale's host starts `zig-ast --serve` at a run's first Zig call and keeps it
for the run. It checks a call's arguments, reads the files, sends them, and
acts on the answer: the values, the stops, the step charges, and the edits,
which it writes through its own `edit`, so they can be undone. The C side of
the wire is Shale's `c/host/zig_wire.c` and `c/host/zig_ast.c`. Shale's
cookbook has recipes for Zig codebases built on these builtins
([cookbook/zig](https://github.com/showell/shale/tree/master/cookbook/zig)).

With Shale, `shale install.shale` in this checkout builds it with the pinned
Zig, found at `tools:zig-<version>/zig`, and installs it as
`tools:zig-ast/bin/zig-ast`, where Shale's host looks. `SHALE_ZIG_AST` names
another binary for one run, as Shale's tests do.

## History

zig-ast grew up inside Shale's repository and moved here on 2026-11-18,
carrying the history of its own folder. Its earlier history is in Shale
(`git log --follow` on Shale's `legacy_zig/src/shale_zig_wire.zig`).

## Files

| file | what it holds |
|---|---|
| `main.zig` | the server: each command, its request and its answer's events |
| `wire.zig` | the wire format, its one home |
| `zig_ast.zig` | the reader: declarations, tokens, identifiers |
| `plan.zig` | the rename planner |
| `zig_move.zig` | the mover |
| `zig_ast_test.zig` | the tests |
| `install.shale` | builds it and installs it where Shale's host looks |
| `zig-version` | the Zig it is built with |

## License

The Universal Permissive License 1.0 ([LICENSE](LICENSE)).
