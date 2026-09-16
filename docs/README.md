# DelphiAST — architecture notes

DelphiAST turns one Delphi or Free Pascal source file into a tree of
`TSyntaxNode` objects. It has no symbol table, resolves nothing across unit
boundaries, and never writes source back out. Understanding how it is built
matters mostly when you want to do something the XML output cannot express —
attach comments to declarations, recover from an incomplete edit, or map nodes
back to exact character ranges.

These pages describe the machinery, not the API surface. They were written by
reading the source, and every claim about behaviour was either traced to a
specific line or measured with a probe program.

## The pages

| Page | What it covers |
|---|---|
| [architecture.md](architecture.md) | The layers, the data flow of one parse, and the inheritance roles the whole design rests on |
| [lexer.md](lexer.md) | `TmwBasePasLex`: character dispatch tables, the buffer stack, conditional compilation, include files |
| [parser.md](parser.md) | `TmwSimplePasPar`: recursive descent, one virtual per production, lookahead, error signalling |
| [tree-builder.md](tree-builder.md) | `TPasSyntaxTreeBuilder`: the node stack, the push/pop discipline, and every place the tree is reshaped |
| [syntax-tree.md](syntax-tree.md) | `TSyntaxNode` and its three subclasses, node types, attributes, ownership, the XML and binary writers |
| [project-indexer.md](project-indexer.md) | `TProjectIndexer`: following a `.dpr` across a search path |
| [extending.md](extending.md) | Consuming the tree, and subclassing the builder without fighting it |
| [limitations.md](limitations.md) | What the design cannot represent, and the defects that follow from it |

## Where to start

If you are consuming the tree, read [syntax-tree.md](syntax-tree.md) and
[limitations.md](limitations.md), in that order. The limitations page is not an
appendix — several of the entries there will change how you design against the
library.

If you are changing the parser, read [architecture.md](architecture.md) then
[tree-builder.md](tree-builder.md), and expect to spend most of your time in
`DelphiAST.pas`.

## A note on this fork

This is De Novo Research's fork of
[RomanYankovsky/DelphiAST](https://github.com/RomanYankovsky/DelphiAST). Two
changes here are not upstream, and are marked **[fork]** where they come up:

- **Procedural-type variants.** `procedure of object` and
  `reference to procedure` now carry a `kind` attribute distinguishing them
  from a plain procedural type. Filed upstream as PR #348.
- **`class var` / `class const`.** The `class` prefix on a var or const section
  is now recorded on the member rather than leaking onto the enclosing type.
  Deliberately not filed: a correct upstream fix has been open as PR #215
  since 2017.

Everything else describes stock DelphiAST.
