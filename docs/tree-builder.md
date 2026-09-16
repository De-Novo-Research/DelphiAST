# The tree builder

`DelphiAST.pas`, class `TPasSyntaxTreeBuilder`. 184 overridden productions and
one stack.

## `TNodeStack`

The stack holds the chain of nodes from the root down to whatever is currently
being parsed. Its top is the node that new children attach to.

| Method | Effect |
|---|---|
| `Push(Typ)` | Creates a node, adds it as a child of the current top, **and pushes it**. Returns it. |
| `Push(Node)` | Pushes an existing node. Used for scratch nodes that are not in the tree. |
| `PushCompoundSyntaxNode(Typ)` | As `Push`, but creates a `TCompoundSyntaxNode` (one that can carry an end position). |
| `PushValuedNode(Typ, Value)` | As `Push`, but a `TValuedSyntaxNode`. |
| `AddChild(Typ)` | Creates a node under the current top and returns it **without** pushing. For leaves. |
| `AddValuedChild(Typ, Value)` | The same, valued. |
| `Peek`, `Pop`, `Clear`, `Count` | As expected. |

Every creation path stamps the node from the lexer:

```pascal
procedure AssignLexerPositionToNode(const Lexer: TPasLexer; const Node: TSyntaxNode);
begin
  Node.LineSeq := Lexer.PosXY.LineSeq;
  Node.Col := Lexer.PosXY.X;
  Node.Line := Lexer.PosXY.Y;
  Node.FileName := Lexer.FileName;
end;
```

So a node's position is *where the lexer was when the node was created*, which
for a bracketing override is the first token of the production. (`LineSeq` is
garbage — see [lexer.md](lexer.md#a-defect-ttokenpointlineseq-is-never-set).)

The stack is public enough to be useful from a descendant: `FStack` is
`protected`, and `Peek` is public, which is what makes comment attachment
possible. See [extending.md](extending.md#attaching-comments-to-declarations).

## The three override idioms

Nearly every one of the 184 overrides is one of these.

**Bracket a subtree.** The node owns everything the production parses.

```pascal
procedure TPasSyntaxTreeBuilder.CaseStatement;
begin
  FStack.Push(ntCase);
  try
    inherited;
  finally
    FStack.Pop;
  end;
end;
```

**Emit a leaf.** No push, so the production's own children (if any) stay with
the parent.

```pascal
procedure TPasSyntaxTreeBuilder.Identifier;
begin
  FStack.AddChild(ntIdentifier).SetAttribute(anName, Lexer.Token);
  inherited;
end;
```

**Decorate the node already on top.** The production is a modifier, not a
structure.

```pascal
procedure TPasSyntaxTreeBuilder.ClassProcedureHeading;
begin
  FStack.Peek.SetAttribute(anKind, AttributeValues[atProcedure]);
  inherited;
end;
```

The third idiom is where most of the subtlety lives, because it depends on
*which* node happens to be on top — and that is a property of the grammar's
call order, not of anything local. The `class var` defect described below is
exactly this going wrong.

## Compound nodes and end positions

`TCompoundSyntaxNode` adds `EndLine` / `EndCol`, set by:

```pascal
procedure TPasSyntaxTreeBuilder.SetCurrentCompoundNodesEndPosition;
begin
  Temp := TCompoundSyntaxNode(FStack.Peek);
  Temp.EndCol := Lexer.PosXY.X;
  Temp.EndLine := Lexer.PosXY.Y;
  Temp.FileName := Lexer.FileName;
end;
```

It is always called **after** `inherited`, at which point the lexer has already
consumed the production's last token and moved to the next one. So the end
position is not the end of the construct — it is the beginning of whatever
follows. Thirteen sites create a compound node, covering eight node types:
`ntInterface`, `ntImplementation`, `ntInitialization`, `ntFinalization`,
`ntUses`, `ntMethod`, `ntTypeDecl` and `ntStatements`. Everything else is a
plain `TSyntaxNode` with a begin position only.

Treat `end_line` / `end_col` as "where the next sibling starts" and nothing
more. See [limitations.md](limitations.md#positions).

## Where the tree is reshaped

The grammar's shape is not always the shape you want, and a handful of
productions rearrange what the recursion produced. The technique is the same
each time: push
a **scratch node** that is not in the tree, let `inherited` fill it, then read
it and emit the real nodes.

### Expressions

`Expression`, `ConstantExpression`, `ThenExpression`, `ElseExpression` and
friends all route through `BuildExpressionTree`. Operand and operator nodes
arrive as a flat list, in source order, because `AdditiveOperator` and its
relatives are leaf emitters:

```pascal
procedure TPasSyntaxTreeBuilder.AdditiveOperator;
begin
  case TokenID of
    ptMinus: FStack.AddChild(ntSub);
    ptOr:    FStack.AddChild(ntOr);
    ptPlus:  FStack.AddChild(ntAdd);
    ptXor:   FStack.AddChild(ntXor);
  end;
  inherited;
end;
```

`TExpressionTools` then runs the shunting-yard algorithm over that list:

1. `PrepareExpr` inserts the invisible operators. A `(` directly after an
   operand becomes an `ntCall`; a `<` type-argument list after an operand
   becomes an `ntGeneric`. It also drops `ntAlignmentParam`.
2. `ExprToReverseNotation` converts to RPN using the priority and associativity
   table at the top of `DelphiAST.Classes.pas` — ten priority levels from
   `ntAddr`/`ntDeref` at 1 down to the relational operators at 9. The
   `ntRoundOpen` / `ntRoundClose` marker nodes are consumed and freed here, so
   **no parenthesis node survives into the finished tree**. Grouping itself is
   not lost: `ExpressionList` has already wrapped the parenthesised text in an
   `ntExpressions` node, so `(A + B) * C` comes out as `MUL` over
   `EXPRESSIONS > EXPRESSION > ADD` and `IDENTIFIER C`.
3. `NodeListToTree` pops the RPN list into a tree, giving unary operators one
   child and binary operators two.

Failures anywhere in this are wrapped as `EParserException` positioned at the
expression's root.

### Statements

`SimpleStatement` collects the statement into a scratch `ntStatement`, then
looks for an `ntAssign` among the children. If there is one, it emits
`ntAssign` with `ntLHS` and `ntRHS` children, each built by running the
expression machinery over the corresponding slice of the list. If there is
none, the whole thing is an `ntCall`.

### Parameter lists

`BuildParametersList` explodes grouped parameters. `procedure Foo(A, B: Integer)`
parses as one group with two names and one type; the builder emits one
`ntParameter` per name, each with a **clone** of the shared type node and of the
default-value expression if present. The parameter's position comes from its
own name node, but the type node inside it carries the position of the single
shared type in the source.

### Var, const and field sections

`RearrangeVarSection` and `ClassField` do the same for `var A, B: Integer` and
for class fields: one `ntVariable` / `ntField` per name, cloned type node
inside each. `ClassField` additionally re-attaches `ntTypeArgs` under the
cloned type.

### Visibility sections

Class members parse as siblings of the `ntPrivate` / `ntPublic` / ... nodes
rather than as their children, because visibility is a marker in the grammar,
not a container. `MoveMembersToVisibilityNodes` runs at the end of `ClassType`
and re-parents each member under the most recent preceding visibility node:

```pascal
if child.HasAttribute(anVisibility) then
  vis := child
else if Assigned(vis) then
begin
  TypeNode.ExtractChild(child);
  vis.AddChild(child);
end;
```

Members declared before any visibility keyword stay directly under the type
node. Repeated sections produce repeated container nodes in source order —
`private ... public ... private ...` gives three containers, not two.

### Dotted names

`TypeId` flattens a nested chain of `ntType` nodes into a single `name`
attribute, `System.SysUtils.TStringList` and all. `UnitName` and
`UsedUnitName` do the same through `NodeListToString`, which joins child
`name` attributes with dots.

## Attributes

Attribute *names* are the 14 values of `TAttributeName`. Attribute *values* are
plain strings, and the common ones come from a lookup table built by RTTI at
unit initialisation:

```pascal
TAttributeValue = (atAsm, atTrue, atFunction, atProcedure, atClassOf, atClass,
  atConst, atConstructor, atDestructor, atEnum, atInterface, atNil, atNumeric,
  atOut, atPointer, atName, atString, atSubRange, atVar, atDispInterface,
  atOfObject, atReferenceTo);

// InitAttributeValues strips the 'at' prefix and lower-cases:
//   atOfObject -> 'ofobject'
```

Storage is a linear `TArray<TAttributeEntry>` per node with linear search.
Nodes rarely have more than three attributes, so this beats a dictionary.

## Comments

Comments are junk to the parser, so they never reach a production. The lexer
raises `OnComment` instead, and the builder catches it:

```pascal
procedure TPasSyntaxTreeBuilder.DoOnComment(Sender: TObject; const Text: string);
begin
  case TokenID of
    ptAnsiComment:    Node := TCommentNode.Create(ntAnsiComment);
    ptBorComment:     Node := TCommentNode.Create(ntBorComment);
    ptSlashesComment: Node := TCommentNode.Create(ntSlashesComment);
  else
    raise EParserException.Create(...);
  end;
  AssignLexerPositionToNode(Lexer, Node);
  Node.Text := Text;
  FComments.Add(Node);
end;
```

The result is a **flat list in source order**, not part of the tree, reachable
through the `Comments` property and destroyed with the builder. The XML writer
ignores it entirely.

What makes this recoverable is that `DoOnComment` fires *during* the parse:
`FStack.Peek` at that moment is the enclosing node. A descendant can attach
comments to declarations deterministically instead of guessing from line gaps.
That technique is written up in
[extending.md](extending.md#attaching-comments-to-declarations).

Comments inside inactive conditional branches are not reported at all — the
lexer gates `DoOnComment` on `FDefineStack = 0`.

## Two known defects in this layer

**`class var` and `class const` mark the wrong node** (fixed in this fork).
`ClassClass` fires when `class` is seen, and writes to `FStack.Peek`. For a
method or property that node is the member, which is correct. For `class var`,
no member node exists yet — the section has not been parsed — so the attribute
lands on the enclosing type's `ntType` node. The result is that class fields
are byte-identical to instance fields in the tree and the class itself gains a
spurious `class="true"`.

**[fork]** This fork records the prefix instead, and lets the member consume it:

```pascal
procedure TPasSyntaxTreeBuilder.ClassClass;
begin
  if FStack.Peek.Typ in [ntMethod, ntProperty] then
    FStack.Peek.SetAttribute(anClass, AttributeValues[atTrue])
  else
    FClassPrefix := True;
  inherited;
end;
```

`ClassField` and `ConstantDeclaration` apply it; `ClassMethodOrProperty` clears
it per member. Upstream has had a correct fix open as PR #215 since 2017.

**Procedural type variants were indistinguishable** (fixed in this fork).
`procedure of object` used to parse byte-identically to a plain `procedure`
type, and `reference to procedure` produced no `ntType` node at all. These are
three different types to the compiler, so the gap was a correctness problem,
not a cosmetic one. **[fork]** `ProceduralDirectiveOf` and
`AnonymousMethodType` now set `kind="ofobject"` and `kind="referenceto"`, and a
new `AnonymousMethodKind` virtual in `SimpleParser.pas` records whether the
anonymous type was a procedure or a function. Filed upstream as PR #348.
