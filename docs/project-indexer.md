# The project indexer

`DelphiAST.ProjectIndexer.pas`, class `TProjectIndexer`.

DelphiAST parses one file. `TProjectIndexer` is the thin driver that turns
that into "parse a `.dpr` and everything it reaches". It is a convenience, not
an analysis layer: it resolves *file paths*, not symbols, and the trees it
produces are exactly the trees `TPasSyntaxTreeBuilder` would have produced one
at a time.

## Use

```pascal
indexer := TProjectIndexer.Create;
try
  indexer.SearchPath := '..\Common;..\Lib;$(BDS)\source\rtl\sys';
  indexer.Defines := 'DEBUG;MYFEATURE';
  indexer.Index('C:\Projects\App\App.dpr');

  for unitInfo in indexer.ParsedUnits do
    if not unitInfo.HasError then
      Walk(unitInfo.SyntaxTree);

  for problem in indexer.Problems do
    Log(problem.FileName, problem.Description);
finally
  indexer.Free;
end;
```

## What `Index` does

1. Clears state, takes the project folder as the implicit first search path.
2. `PrepareDefines` splits `Defines` on the delimiter into a list;
   `PrepareSearchPath` expands the path list relative to the project folder.
3. Registers `<project>.dpr` in the unit-path cache and parses it.
4. `ScanUsedUnits` reads the uses clauses out of the resulting tree —
   `BuildUsesList` walks both the interface and implementation `ntUses`
   nodes — and recurses into each unit not yet parsed.
5. Fills `ParsedUnits`, `IncludeFiles`, `Problems` and `NotFoundUnits`.

Each unit is parsed with its own builder, seeded with the compiler's defines
(unless you clear `piUseDefinesDefinedByCompiler` from `Options`) plus yours:

```pascal
if piUseDefinesDefinedByCompiler in Options then
  builder.InitDefinesDefinedByCompiler;
for define in FDefinesList do
  TmwSimplePasPar(builder).Lexer.AddDefine(define);
```

Note the ordering and the cast — this is the supported way to push project
defines in, and it matters, because without it every unit is parsed under
*your* compiler's version defines. See
[lexer.md](lexer.md#conditional-compilation).

## Unit resolution

`FindFile` looks for `<name>.pas` in the folder of the referencing file, then
along the search path. A `uses Foo in 'sub\Foo.pas'` clause in a `.dpr`
overrides that, using the `anPath` attribute the builder records on the
`ntUnit` node.

Units that cannot be found land in `NotFoundUnits` and are not an error — this
is how the RTL is normally handled: don't put it on the search path, and the
indexer stops at your own code.

## Includes

The indexer supplies its own `IIncludeHandler`, backed by a cache keyed on
resolved path, so a `.inc` pulled in by forty units is read once. Every include
actually consumed shows up in `IncludeFiles`.

## Errors and interruption

Parse failures are captured, not raised. `ESyntaxTreeException` is caught per
unit, recorded in `Problems` as `ptCantParseFile` with line and column, and the
unit is marked `HasError` — with whatever partial tree survived. Indexing
continues.

`TProblemType` covers `ptCantFindFile`, `ptCantOpenFile` and `ptCantParseFile`.

Two events let you interpose:

- `OnGetUnitSyntax` fires before a unit is parsed. Supply a `syntaxTree`
  yourself and set `doParseUnit := False` to serve it from your own cache.
- `OnUnitParsed` fires after, with `syntaxTreeFromParser` telling you which of
  the two happened. You may replace the tree — take ownership carefully.

Either can set `doAbort` to stop the walk.

## What it does not do

No symbol table, no cross-unit name resolution, no dependency ordering beyond
the traversal order, and no caching between runs. If you want to know which
unit declares `TFoo`, you walk the trees yourself.
