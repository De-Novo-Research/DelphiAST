# Consuming and extending

## Reading a tree

```pascal
uses
  DelphiAST, DelphiAST.Classes, DelphiAST.Consts;

procedure ListMethods(const aFileName: string);
var
  root, intf, node: TSyntaxNode;
begin
  root := TPasSyntaxTreeBuilder.Run(aFileName);
  try
    intf := root.FindNode(ntInterface);
    if Assigned(intf) then
      for node in intf.ChildNodes do
        if node.Typ = ntMethod then
          WriteLn(node.GetAttribute(anName), ' at line ', node.Line);
  finally
    root.Free;
  end;
end;
```

`GetAttribute` returns `''` for an absent attribute rather than raising, so
`HasAttribute` is only needed when empty and absent must be told apart. There is
no visitor base class; recursion over `ChildNodes` is the idiom.

## Which `Run` to call

```pascal
// class method - simplest, and frees the builder for you
class function Run(const FileName: string; InterfaceOnly: Boolean = False;
  IncludeHandler: IIncludeHandler = nil;
  OnHandleString: TStringEvent = nil): TSyntaxNode; static;

// instance method - use this when you need anything the class method hides
function Run(SourceStream: TStream): TSyntaxNode;
```

Use the instance method when you need to parse text you already hold rather
than a file, push project defines in, or read `Comments` afterwards — the class
method frees the builder, and the comments with it.

```pascal
builder := TPasSyntaxTreeBuilder.Create;
try
  builder.InterfaceOnly := False;
  builder.IncludeHandler := TMyIncludes.Create(projectFolder);
  builder.InitDefinesDefinedByCompiler;
  TmwSimplePasPar(builder).Lexer.AddDefine('MYFEATURE');

  root := builder.Run(sourceStream);
  try
    UseComments(builder.Comments);
    Walk(root);
  finally
    root.Free;
  end;
finally
  builder.Free;
end;
```

The cast to `TmwSimplePasPar` is needed because `TmwSimplePasParEx` redeclares
`Lexer` as the narrow `TPasLexer` facade. `TProjectIndexer` does exactly this.

## Include handler

Without one, `{$I}` is skipped silently and the included declarations are
missing from the tree with no warning.

```pascal
type
  TMyIncludes = class(TInterfacedObject, IIncludeHandler)
  private
    FRoot: string;
  public
    constructor Create(const aRoot: string);
    function GetIncludeFileContent(const aParentFileName, aIncludeName: string;
      out aContent, aFileName: string): Boolean;
  end;
```

Return `False` when the file cannot be found; the lexer carries on without it.
The `FileName` you return is stamped on every node that comes from the include,
and positions inside it are relative to that file, not to the including unit.

## Interning identifier strings

```pascal
pool := TStringPool.Create;
builder.OnHandleString := pool.StringIntern;
```

Every identifier, type name and dotted name passes through the hook on its way
into the tree. Across a project where `Integer` occurs fifty thousand times,
this collapses those into one string. Keep the pool alive as long as any tree
that used it.

## Subclassing the builder

The point of the design is that `TPasSyntaxTreeBuilder` is not the only
possible consumer of the parse. 341 productions are `virtual`; 184 are already
overridden, and the rest are free.

```pascal
type
  TMyBuilder = class(TPasSyntaxTreeBuilder)
  protected
    procedure ClassMethodHeading; override;
  end;

procedure TMyBuilder.ClassMethodHeading;
begin
  inherited;                       // the node has been built and popped
  Inc(FMethodCount);
end;
```

Five rules, learned the hard way:

1. **Always call `inherited`.** Skipping it skips the entire sub-production —
   the parser will not advance past those tokens and the next production will
   see them instead.
2. **Decide where your work goes relative to `inherited`.** Before it, the
   lexer is on the production's first token and the node does not exist yet.
   After it, the node has usually been popped already and the lexer has moved
   past the construct.
3. **Keep the stack balanced.** `Run` asserts `FStack.Count = 0` at the end.
   Use `try..finally` around every `Push`.
4. **Do not assume what `FStack.Peek` is** in a decorate-style override. It is
   whatever the grammar's call order left on top, and that is not always the
   node you have in mind. The upstream `class var` defect is exactly this
   mistake — see [tree-builder.md](tree-builder.md#two-known-defects-in-this-layer).
5. **Speculative parses never reach you.** Lookahead runs on a separate
   `TmwSimplePasPar` instance, not on your class, so nothing you override can
   fire from a path the parser was only trying out.

### What you cannot hook

Several interesting methods are `private` in `TPasSyntaxTreeBuilder` and cannot
be overridden: `ParserMessage`, `DoOnComment`, `BuildExpressionTree`,
`BuildParametersList`, `RearrangeVarSection`, `MoveMembersToVisibilityNodes`.
Changing any of those behaviours means either overriding the virtual `Run` and
reimplementing it, or editing the unit.

`FStack` and `FComments` are `protected`, so a descendant can read and
manipulate both.

## Attaching comments to declarations

Comments never reach a production, so there is no `Comment` virtual to
override. But `OnComment` is a public property on the parser, and the lexer
raises it **during** the parse — at which point `FStack.Peek` is the node
currently being built.

```pascal
constructor TMyBuilder.Create;     // declared 'override' - the base is virtual
begin
  inherited;
  OnComment := HandleComment;      // replaces the base handler
end;

procedure TMyBuilder.HandleComment(Sender: TObject; const aText: string);
begin
  FPending.Add(TPendingComment.Create(aText, Lexer.PosXY, FStack.Peek));
end;
```

Replacing the handler means `Comments` is no longer filled, so record whatever
you need yourself.

What `Peek` gives you is the **enclosing** node, not the declaration the comment
documents — the declaration has not been parsed yet when its doc comment is
scanned. So the usable rule is: a comment scanned while node *N* was on top,
followed by a child of *N* created at a later position, documents that child.
That is still a great deal better than reconciling a flat comment list against
the tree by line arithmetic afterwards, which is the only option if you consume
the XML.

## Going one layer deeper

If `TSyntaxNode` is not the shape you want, you do not have to take it.
Descending `TmwSimplePasParEx` directly and overriding the productions you care
about gives you the same hooks with none of DelphiAST's tree-building — useful
for a metrics counter, a formatter, or a model of your own that would otherwise
be built by walking and discarding a `TSyntaxNode` tree.

The cost is real: the 184 overrides in `DelphiAST.pas` encode a lot of
accumulated knowledge about where the grammar's shape and a useful tree's shape
differ, and you would be rediscovering it. Descending
`TPasSyntaxTreeBuilder` and keeping its tree is the cheaper route unless you
have measured a reason not to.
