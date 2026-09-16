# The syntax tree

`DelphiAST.Classes.pas` and `DelphiAST.Consts.pas`.

## `TSyntaxNode`

One class, one type tag, an attribute list and a child array. There is no
`TIfNode` or `TMethodNode`; a node's meaning is entirely in `Typ`.

```pascal
TSyntaxNode = class
public
  property Typ: TSyntaxNodeType;
  property ParentNode: TSyntaxNode;
  property ChildNodes: TArray<TSyntaxNode>;
  property Attributes: TArray<TAttributeEntry>;

  property Line: Integer;
  property Col: Integer;
  property FileName: string;
  property LineSeq: Integer;     // never populated - see lexer.md
end;
```

Three subclasses add one field each:

| Class | Adds | Used for |
|---|---|---|
| `TCompoundSyntaxNode` | `EndLine`, `EndCol` | Sections, methods, type declarations, statement blocks |
| `TValuedSyntaxNode` | `Value: string` | Literals, `ntName` where the text matters |
| `TCommentNode` | `Text: string` | Comments — which live outside the tree |

`Clone` is deep and preserves the class through
`TSyntaxNodeClass(Self.ClassType).Create(FTyp)`, so cloning a compound node
gives a compound node. The builder relies on this when it explodes grouped
declarations.

### Storage

`FChildNodes` and `FAttributes` are plain `TArray<T>`, and `AddChild` /
`SetAttribute` grow them by exactly one element:

```pascal
SetLength(FChildNodes, Length(FChildNodes) + 1);
FChildNodes[Length(FChildNodes) - 1] := Node;
```

That is a reallocation per child. It is a deliberate trade — nodes with many
children are rare, and the memory saved against a `TList<T>` per node across
hundreds of thousands of nodes is large. Attribute lookup is a linear scan for
the same reason.

### Navigation

`FindNode(Typ)` returns the first direct child of that type, or nil.

`FindNode(const TypesPath: array of TSyntaxNodeType)` walks a path, with
`ntUnknown` as a wildcard for any type. For this branch:

```xml
<VARIABLE line="9" col="3">
  <NAME line="9" col="3" value="ValueRec"/>
  <TYPE line="9" col="13" name="LongInt"/>
  <ABSOLUTE line="9" col="21">
    <VALUE line="9" col="30">
      <EXPRESSION line="9" col="30">
        <IDENTIFIER line="9" col="30" name="AValue"/>
```

all three of these find the identifier:

```pascal
FindNode([ntAbsolute, ntValue, ntExpression, ntIdentifier])
FindNode([ntAbsolute, ntUnknown, ntExpression, ntIdentifier])
FindNode([ntAbsolute, ntUnknown, ntUnknown, ntIdentifier])
```

There is no search by name, no XPath and no visitor base class. Walking is
`for child in Node.ChildNodes do` and recursion.

### Mutation

`AddChild`, `DeleteChild` (extract and free) and `ExtractChild` (unlink,
caller owns) are all public, and `ClearAttributes` exists. The tree is
genuinely mutable, which is what lets `MoveMembersToVisibilityNodes` re-parent
class members after the fact.

## Node types

136 values in `TSyntaxNodeType`, each with a lower-case string name in
`SyntaxNodeNames` used for XML output. Roughly grouped:

| Group | Values |
|---|---|
| File structure | `ntUnit`, `ntPackage`, `ntInterface`, `ntImplementation`, `ntInitialization`, `ntFinalization`, `ntUses`, `ntContains`, `ntRequires`, `ntExports` |
| Declarations | `ntTypeSection`, `ntTypeDecl`, `ntType`, `ntVariables`, `ntVariable`, `ntConstants`, `ntConstant`, `ntResourceString`, `ntField`, `ntFields`, `ntMethod`, `ntProperty` |
| Type structure | `ntEnum`, `ntSubrange`, `ntBounds`, `ntDimension`, `ntGuid`, `ntHelper`, `ntTypeParams`, `ntTypeParam`, `ntTypeArgs`, `ntConstraints` and the three constraint kinds |
| Visibility | `ntPrivate`, `ntProtected`, `ntPublic`, `ntPublished`, `ntStrictPrivate`, `ntStrictProtected` |
| Signatures | `ntParameters`, `ntParameter`, `ntReturnType`, `ntName`, `ntExternal`, `ntMessage`, `ntIndex`, `ntRead`, `ntWrite`, `ntDefault`, `ntImplements`, `ntResolutionClause` |
| Statements | `ntStatements`, `ntStatement`, `ntIf`, `ntThen`, `ntElse`, `ntCase` and its four helpers, `ntFor`/`ntTo`/`ntDownTo`/`ntFrom`, `ntWhile`, `ntRepeat`, `ntWith`, `ntTry`/`ntExcept`/`ntFinally`/`ntExceptionHandler`, `ntRaise`, `ntGoto`, `ntLabel`, `ntAssign`, `ntCall`, `ntInherited`, `ntEmptyStatement` |
| Expressions | `ntExpression`, `ntExpressions`, `ntLHS`, `ntRHS`, `ntLiteral`, `ntIdentifier`, `ntSet`, `ntElement`, `ntValue`, `ntTernaryOp`, `ntAnonymousMethod` |
| Operators | 30 of them, from `ntAddr` and `ntDeref` through `ntAdd`, `ntMul`, `ntDot`, `ntCall`, `ntGeneric`, `ntIndexed` to `ntIs`, `ntIsNot`, `ntIn`, `ntNotIn` |
| Attributes (Delphi `[...]`) | `ntAttributes`, `ntAttribute`, `ntNamedArgument`, `ntPositionalArgument` |
| Comments | `ntAnsiComment`, `ntBorComment`, `ntSlashesComment` — used by `TCommentNode` only |
| Sentinel | `ntUnknown` — also the wildcard in `FindNode` |

`ntRoundOpen` and `ntRoundClose` exist but are transient: the expression
builder consumes and frees them. You will never see one in a finished tree.

## Attributes

14 names, all values plain strings.

| Attribute | Appears on | Typical values |
|---|---|---|
| `anName` | Most declaration and reference nodes | the identifier, or a dotted name |
| `anType` | `ntType`, `ntStatements`, `ntLiteral` | `class`, `classof`, `interface`, `dispinterface`, `enum`, `subrange`, `pointer`, `string`, `numeric`, `nil`, `asm` |
| `anKind` | `ntMethod`, `ntParameter`, `ntType` | `procedure`, `function`, `constructor`, `destructor`, `const`, `var`, `out`, `ofobject` **[fork]**, `referenceto` **[fork]** |
| `anVisibility` | The six visibility nodes | `true` |
| `anClass` | `ntMethod`, `ntProperty`, `ntField` **[fork]**, `ntConstant` **[fork]** | `true` |
| `anForwarded` | `ntType` | `true` |
| `anCallingConvention` | `ntMethod` | `stdcall`, `cdecl`, `register`, ... |
| `anMethodBinding` | `ntMethod` | `virtual`, `dynamic`, `override`, ... |
| `anReintroduce`, `anOverload`, `anAbstract`, `anInline` | `ntMethod` | `true` |
| `anAlign` | record types | the alignment value |
| `anPath` | `ntUnit` in a `.dpr` uses clause | the `in 'Foo.pas'` file name |

## Output formats

### XML

`TSyntaxTreeWriter.ToXML(Root, Formatted)`. Element name is the upper-cased
node-type name; position attributes are `line`/`col` for a plain node and
`begin_line`/`begin_col`/`end_line`/`end_col` for a compound one; `value=` for
a valued node; `file=` when the node came from an include file; then the node's
own attributes.

Three things to know:

- **`line_seq=` is emitted on every element and is garbage.** See
  [lexer.md](lexer.md#a-defect-ttokenpointlineseq-is-never-set).
- **Comments are not in the output at all.** They are not in the tree, and the
  writer does not consult `Comments`.
- The XML is a *view*, not a serialisation — it cannot be read back. Use the
  binary format for that.

### Binary

`TSyntaxTreeWriter.ToBinary(Root, Stream)` and `TBinarySerializer.Read`.
Round-trips the tree including node classes and their extra fields. The format
is a `'DAST binary file'#26` signature, a version word checked as
`(version and $FFFF0000) = $01000000`, then nodes depth-first with strings
deduplicated through a stream-local string table and numbers written as
variable-length integers. Not available under FPC.
