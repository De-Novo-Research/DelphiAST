# Limitations

Everything here was checked against the source and, where an example is shown,
reproduced with a small probe program against this fork's `master`. Some of
these are design decisions, some are defects. Both matter equally if you are
building on the library.

## No symbol table

Stated in the README, but worth restating because it shapes everything else. An
`ntIdentifier` node is a name, a line and a column. DelphiAST does not know
whether it is a variable, a method, a unit or a type, does not resolve
overloads, and does not connect a use to its declaration — not even within the
same unit. Anything of that kind is yours to build on top.

## Conditional compilation is resolved, not recorded

The lexer evaluates `{$IFDEF}` and makes every token in a dead branch
[junk](lexer.md#junk-and-why-the-parser-never-sees-whitespace). There is no
conditional node type. The tree you get is **one configuration**, with no
record that any other existed:

```pascal
{$IFDEF NOT_DEFINED_ANYWHERE}
procedure OnlyInThatBranch;
{$ELSE}
procedure TheOtherOne;
{$ENDIF}
```

```xml
<INTERFACE begin_line="2" begin_col="1" end_line="8" end_col="1">
  <METHOD begin_line="6" begin_col="1" end_line="8" end_col="1"
          kind="procedure" name="TheOtherOne"/>
</INTERFACE>
```

`OnlyInThatBranch` is not in the tree, not in a disabled subtree, and not
mentioned anywhere. For a browser this is a missing declaration; for a
refactoring tool it is a correctness hazard, because a rename computed from the
tree silently misses every occurrence in the branch that was not taken.

Three compounding problems:

- **The default defines are the host compiler's.** `InitDefinesDefinedByCompiler`
  is evaluated when *your* binary is built, so a unit written for XE7 is parsed
  under your Delphi's `VER` and `CompilerVersion`. Push the target project's
  defines in with `Lexer.AddDefine`.
- **`{$IF}` evaluates almost nothing.** Only `CompilerVersion`/`RTLVersion`
  compared against an integer, and flat `Defined()` chains. Anything else is
  **False, with no diagnostic** — so the `{$ELSE}` branch is taken silently.
- **`{$IFOPT}` is unconditionally False.**

## Error recovery

There is none, in the usual sense. The first error ends the parse, and what you
get back is a **prefix** of the unit — everything up to the failure point and
nothing after it.

```pascal
unit Sample;
interface
type
  TFoo = clas            // mid-edit typo
    procedure Bar;
  end;
implementation
procedure TFoo.Bar;
begin
end;
procedure Other;
begin
end;
end.
```

```xml
<UNIT line="1" col="1" name="Sample">
  <INTERFACE begin_line="2" begin_col="1" end_line="0" end_col="0">
    <TYPESECTION line="3" col="1">
      <TYPEDECL begin_line="4" begin_col="3" end_line="5" end_col="5" name="TFoo">
        <TYPE line="4" col="10" name="clas"/>
      </TYPEDECL>
    </TYPESECTION>
  </INTERFACE>
</UNIT>
```

The entire implementation section is gone — two perfectly well-formed methods,
absent because of one missing character forty lines earlier. Note also that the
interface's `end_line` is `0`, because the section never finished. For a tool
that re-parses as the user types, this means the outline empties below the
cursor whenever a declaration is momentarily incomplete. `begin..end` pairing
does **not** let the parser resynchronise at method boundaries.

Mechanically the prefix arrives by exception, not by return value:

```pascal
try
  tree := builder.Run(stream);
except
  on e: ESyntaxTreeException do
  begin
    tree := e.SyntaxTree;
    e.SyntaxTree := nil;   // or the exception's destructor frees it
  end;
end;
```

Two mitigations work in practice:

- **`InterfaceOnly := True`.** Method bodies are never entered, so a unit with
  broken bodies still yields a complete declaration tree.
- **Stub and retry.** Blank out the body of the method that failed, reparse,
  repeat. Parsing is fast enough that several attempts are still sub-millisecond
  for a typical unit.

### Related: you cannot install a tolerant message handler

`TPasSyntaxTreeBuilder.Run` assigns `OnMessage` to its own handler on entry
(`DelphiAST.pas:2169`), overwriting whatever you set. And that handler raises on
every `meError`. Worse, `ParserMessage` is `private`, so a descendant cannot
override it either — changing the error policy means overriding the virtual
`Run` and reimplementing what it does. Assigning `OnMessage` from outside
achieves nothing.

### Related: transposed line and column

On the `ESyntaxError` path only — that is, the fatal path reached through
`ExpectedFatal`, typically on unexpected end of file — the re-raise passes the
arguments the wrong way round:

```pascal
on E: ESyntaxError do
  raise ESyntaxTreeException.Create(E.PosXY.X, E.PosXY.Y, ...);
//                                  ^ column   ^ line
```

`ESyntaxTreeException.Create` takes `(Line, Col, ...)`. Errors arriving via
`EParserException` — which is most of them — are correct.

## Positions

**Only begin positions are trustworthy.** Most nodes have `Line` and `Col` and
nothing else. The thirteen sites that create a `TCompoundSyntaxNode` set an end
position *after* parsing the construct, by which point the lexer has already
moved to the next token — so `end_line`/`end_col` is effectively where the next
sibling begins, not where this node ends.

**`LineSeq` is uninitialised memory.** The lexer maintains the counter but
`GetPosXY` no longer copies it out, so every node and every `line_seq=` XML
attribute carries stack garbage. Details and provenance in
[lexer.md](lexer.md#a-defect-ttokenpointlineseq-is-never-set).

**Nodes carry no character offsets.** The lexer has `TokenPos` and `TokenLen`,
which are exact, but nothing propagates them into the tree. If you need precise
source ranges — to splice text, for instance — you have to run your own token
pass over the buffer alongside the parse.

## Literal fidelity is not guaranteed

The number lexer does not handle a signed exponent:

```pascal
const
  FLOOR = 1e-5;
```

```xml
<SUB line="4" col="13">
  <LITERAL line="4" col="11" value="1e" type="numeric"/>
  <LITERAL line="4" col="14" value="5"  type="numeric"/>
</SUB>
```

`1e-5` becomes a subtraction of `1e` and `5`. The parse does not fail, and if
you evaluate or re-emit that subtree you get something that is not the original
constant. Any tool that reads literal values out of the tree should treat them
as approximate and go back to the source text when it matters.

## The tree cannot be printed back to source

There is no Pascal emitter, and `TSyntaxTreeWriter` does XML and binary only.
The tree could not support one anyway, because too much has been discarded by
the time it exists:

- comments are not in it at all
- formatting, blank lines and indentation are gone
- compiler directives are gone
- inactive conditional branches are gone
- literals are not guaranteed faithful, as above
- parenthesis markers are consumed by the expression builder

The practical consequence for a refactoring tool: **produce text edits against
the original buffer**, never regenerated source. Splicing preserves everything
unmodellable for free.

### How other parsers solve this

This is a solved problem elsewhere, and it is worth knowing the vocabulary
before concluding DelphiAST should have solved it too. A tree that can
reproduce its input exactly is called **lossless** or **full fidelity**: the
guarantee is `print(tree) = original source`, character for character. Two
mechanisms are in use.

**Trivia attached to tokens.** Whitespace, comments and directives are not
nodes of their own; each token carries *leading* and *trailing* trivia. The
abstract shape of the tree stays clean — you still walk `IfStatement` to its
condition — but every character in the file hangs off some token, so printing
is a traversal that emits trivia as well. Roslyn coined the term and most
later designs copied it.

**Exact ranges over a retained buffer.** The tree stores byte offsets that
cover the file with no gaps, and reproduction is slicing the source you still
hold. Much cheaper, but it round-trips only while you have the buffer.

| Parser | Approach |
|---|---|
| Roslyn (C#/VB) | Trivia. `ToFullString()` round-trips exactly; even malformed input is kept, as skipped-token trivia |
| rowan / rust-analyzer | Whitespace and comments are ordinary nodes in the CST — lossless by construction |
| SwiftSyntax | Trivia; the Swift parser was reworked specifically to guarantee round-tripping |
| IntelliJ PSI | `PsiWhiteSpace` and `PsiComment` are tree elements, and `getText()` returns the original. This is what IntelliJ's refactorings run on |
| LibCST (Python) | Built because Python's own `ast` discards formatting, which makes codemods unusable |
| tree-sitter | Range-based, with comments as "extras"; every node spans real bytes |

In the Delphi world the clearest example is the **Jedi Code Formatter**. A
formatter cannot work any other way: it tokenises everything, whitespace and
comments included, builds a tree over all of it, and regenerates from that.

**What it costs.** Full fidelity roughly doubles the node count, and every
traversal then has to skip trivia, so the API grows a second set of accessors
for "children that matter". Incremental reparsing becomes worth having too,
which is why Roslyn and rowan both use **red-green trees** — immutable,
width-only "green" nodes that can be shared between versions of a file, under a
lazily constructed "red" layer that supplies parents and absolute positions.
That is a great deal of machinery for a library whose brief is to hand you an
AST for one unit, and DelphiAST reasonably declines it.

**One limit even at the top end.** Roslyn does not give you a parsed tree for
the untaken `#if` branch either — it keeps the text as `DisabledTextTrivia`, so
the file reproduces, but the disabled code cannot be walked as structure.
Analysing every configuration means parsing the file once per configuration.
DelphiAST's
[conditional-compilation gap](#conditional-compilation-is-resolved-not-recorded)
is therefore worse in degree rather than in kind: it does not even retain the
text.

**So the edits-not-regeneration rule is not a workaround for a deficient
parser.** It is what Roslyn users frequently do as well, because editing the
original buffer preserves formatting on every byte you did not touch, for free
and with no risk. The price is that you can only express a change you can
locate exactly — which is what makes `TokenPos` and `TokenLen` failing to reach
the tree ([above](#positions)) more irritating than it first looks.

## Comments are outside the tree

They go to a flat `TPasSyntaxTreeBuilder.Comments` list in source order, owned
by the builder, and the XML writer ignores them. Two consequences:

- The class-method `Run` frees the builder before returning, so comments are
  **unreachable** through that entry point. Use the instance method.
- Anything XMLDoc-shaped must be built against the live tree, not the XML.

Attaching them to declarations reliably is possible, but it needs a descendant
— see [extending.md](extending.md#attaching-comments-to-declarations).

Comments inside an inactive conditional branch are never reported.

## Includes are silently skipped without a handler

`{$I foo.inc}` does nothing unless you supply an `IIncludeHandler`. There is no
warning; the included declarations are simply absent from the tree. If you are
parsing real projects, supplying a handler is not optional.

## Fixed in this fork

Both of these are live defects upstream.

- **Procedural type variants.** `procedure of object` used to parse
  byte-identically to a plain `procedure` type, and `reference to procedure`
  produced no `ntType` node at all. These are three distinct types to the
  compiler, so the gap was a correctness problem. Fixed here; filed upstream as
  PR #348.
- **`class var` / `class const`.** The `class` prefix landed on the enclosing
  type's node rather than the member's, making class fields indistinguishable
  from instance fields and giving the class a spurious `class="true"`. Fixed
  here; deliberately not filed, because a correct fix has been open upstream as
  PR #215 since 2017.

See [tree-builder.md](tree-builder.md#two-known-defects-in-this-layer).
