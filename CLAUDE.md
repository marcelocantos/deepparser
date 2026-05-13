# deepparser

SQL parser for SQLite with sqldeep grammar extensions. Forked from
[sqliteai/liteparser](https://github.com/sqliteai/liteparser); same
Lemon LALR(1) grammar (`src/lp_parse.y`), same tokenizer
(`src/lp_tokenize.c`), same AST and arena model — extended to
recognise sqldeep's JSON-shaped SELECT projections, join arrows,
JSON-path operator, XML literals, and recursive selects.

Used by [sqldeep](https://github.com/marcelocantos/sqldeep) as its
front-line SQL parser. Standalone consumers welcome but not the
primary use case.

## Build

```sh
make            # build static + dynamic libraries + sqlparse CLI
make test       # build and run all tests (currently upstream tests only)
make clean
```

Lemon parser generator is built locally from `lemon.c` (vendored).
`make grammar` regenerates `parse.c` from `src/lp_parse.y`.

## Upstream tracking

- `origin` → `github.com/marcelocantos/deepparser` (our fork).
- `upstream` → `github.com/sqliteai/liteparser` (read-only).
- Pull upstream changes via `git fetch upstream && git merge upstream/main`
  when meaningful commits land. Upstream churn is low (~5 commits over
  ~6 months) and almost entirely additive.

## Architecture

Three concerns, three files:

1. **`lp_tokenize.c`** — hand-written tokenizer. Adapted from SQLite's
   `tokenize.c`. Emits a stream of `LpToken` with source positions.
2. **`lp_parse.y`** — Lemon grammar. Compiled to `parse.c`. Builds
   `LpNode` AST via arena-allocated nodes.
3. **`lp_unparse.c`** — AST → SQL text. Canonical (not byte-identical to
   input). New sqldeep node kinds get matching emitters here.

## Sqldeep extensions

Added on top of the standard SQLite grammar:

- **Object literal** `{ field, key: expr, ... }` — expression.
- **Array literal** `[ expr, ... ]` — expression.
- **Deep SELECT projection** `SELECT { ... }` / `SELECT [ ... ]` /
  `SELECT/1 { ... }` — statement / subquery.
- **FROM-first variant** `FROM ... SELECT { ... }` — alternative
  ordering for deep projections.
- **Join arrow** `c->orders o ON|USING ...`, `<-` for reverse, chains
  and bridges — in FROM-position only.
- **JSON path** `(expr).field.sub[n]` — expression.
- **XML element** `<tag attr="v">body</tag>`, self-closing, namespaced,
  with `{expr}` interpolation. `jsx(...)` and `jsonml(...)` wrappers
  select alternative emit modes (sqldeep concern, see node `mode`
  field).
- **Recursive SELECT** `SELECT/1 { ..., children: * } FROM t RECURSE ON (fk [= pk]) WHERE ...`.
- **`SELECT/1`** singular modifier.

Grammar productions for these live in `src/lp_parse.y` under
sqldeep-tagged sections. Corresponding `LpNodeKind`s have an
`LP_SQLDEEP_` prefix.

Token disambiguation:

- `->` followed by ident → `JOIN_ARROW`; followed by string/expr →
  binary `LP_OP_PTR`.
- `<` followed by ident → `XML_LT`; otherwise comparison `LT`.
- `/` after `SELECT` → `SLASH_ONE` if followed by `1`; otherwise
  divide.
- `[` is sqldeep array start (SQLite bracketed-identifier lexing
  disabled — non-portable, not a sqldeep target).

## Round-trip

Parse → AST → unparse → semantically equivalent text, at parity with
upstream liteparser. **Not byte-identical**: whitespace normalised,
comma-joins expanded to `INNER JOIN`, `i+1` → `i + 1`, etc. This
matches upstream behaviour and is not a regression.

## File layout

```
src/
  arena.[ch]              Arena allocator
  liteparser.[ch]         Public API, AST types, visitor, JSON
  liteparser_internal.h   Internal types
  lp_lempar.c             Lemon parser template (generated)
  lp_parse.y              Grammar source
  lp_tokenize.c           Hand-written tokenizer
  lp_unparse.c            AST → SQL
  parse.[ch]              Lemon-generated parser
test/                     Test suite (upstream + sqldeep additions)
fuzz/                     Fuzz harness
wasm/                     WASM build
Makefile                  Build
```

## Licensing

MIT License throughout, matching upstream. Modifications to upstream
files inherit MIT; new files added in this fork are also MIT. The
LICENSE file carries both the upstream SQLite AI copyright and a
contributor line for sqldeep grammar extensions. See `LICENSE`.

Note: this deviates from the global "always Apache 2.0" convention.
Relicensing upstream MIT code is not an option, and dual-licensing
just the additions would create a per-file licensing minefield that
doesn't pay for itself. Sqldeep itself (the consumer) remains
Apache 2.0; the MIT/Apache 2.0 boundary is at the submodule edge.
