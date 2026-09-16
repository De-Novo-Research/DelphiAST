# The parser

`SimpleParser.pas`, class `TmwSimplePasPar`, extended by
`TmwSimplePasParEx` in `DelphiAST.SimpleParserEx.pas`.

## Shape

Recursive descent, written by hand, one method per grammar production, 341 of
them `virtual`. The methods are named after the Delphi grammar:
`UnitFile`, `InterfaceSection`, `TypeSection`, `TypeDeclaration`, `ClassType`,
`ClassMethodHeading`, `FormalParameterList`, `Statement`, `IfStatement`,
`Expression`, `SimpleExpression`, `Term`, `Factor`, `Designator`.

A production is written as a straight transcription of the grammar rule:

```pascal
procedure TmwSimplePasPar.ProgramFile;
begin
  Expected(ptProgram);
  UnitName;
  if TokenID = ptRoundOpen then
  begin
    NextToken;
    IdentifierList;
    Expected(ptRoundClose);
  end;
  if not InterfaceOnly then
  begin
    Semicolon;
    ProgramBlock;
    Expected(ptPoint);
  end;
end;
```

By itself the parser builds nothing. Run it on its own and the only observable
effects are the `OnMessage` calls it makes on bad input. That is the point:
`TPasSyntaxTreeBuilder` gets a complete, pre-factored set of hooks for free,
and anyone wanting a different output (a metrics counter, a formatter, a
different tree) gets the same hooks.

## Entry

```pascal
procedure TmwSimplePasPar.Run(const UnitName: string; SourceStream: TStream);
```

Copies the stream into a string, assigns it to `FLexer.Origin` (which lexes
the first token as a side effect of the setter), and calls `ParseFile`.
`ParseFile` does `SkipJunk` and dispatches on `GenID` to `LibraryFile`,
`PackageFile`, `ProgramFile` or `UnitFile` — and to `IncludeFile` for anything
else, which is how a bare `.inc` fragment can be parsed.

## Consuming tokens

Three routines do nearly all the work:

| Routine | Behaviour on mismatch |
|---|---|
| `Expected(Sym)` | Reports `meError` through `OnMessage` and **does not advance**. If the current token is `ptNull` (end of file) it escalates to `ExpectedFatal`. |
| `ExpectedEx(Sym)` | The same, but compares against `Lexer.ExID` rather than `TokenID`. |
| `ExpectedFatal(Sym)` | Raises `ESyntaxError` carrying `Lexer.PosXY`. |

On a match, all three call `NextToken`, which is `Lexer.NextNoJunk` — so
whitespace, comments and conditional directives never reach the parser. See
[lexer.md](lexer.md#junk-and-why-the-parser-never-sees-whitespace).

`SynError(TmwParseError)` reports a production-level failure ("this is not a
valid const section") through the same `OnMessage` channel. The error
enumeration is in `SimpleParser.Types.pas`.

## The error model

There are two channels and they behave differently:

- **`OnMessage` with `meError`.** Advisory. The base parser carries on from
  wherever it is. Because `Expected` does not advance on a mismatch, carrying
  on usually means the next production sees the same unexpected token.
- **`ESyntaxError`.** Raised by `ExpectedFatal`, which is reached on
  end-of-file and from a handful of explicit sites. Unwinds the parse.

`TPasSyntaxTreeBuilder` collapses the two by installing a handler that turns
every `meError` into a raised `EParserException`:

```pascal
procedure TPasSyntaxTreeBuilder.ParserMessage(Sender: TObject;
  const Typ: TMessageEventType; const Msg: string; X, Y: Integer);
begin
  if Typ = TMessageEventType.meError then
    raise EParserException.Create(Y, X, Lexer.FileName, Msg);
end;
```

So when you are building a tree, the first error ends the parse. There is no
resynchronisation and no second error. Two consequences worth reading before
designing around them: [limitations.md](limitations.md#error-recovery), and the
fact that `Run` reassigns `OnMessage` on entry, so **installing your own
tolerant handler on the builder has no effect** (`DelphiAST.pas:2169`).

`meNotSupported` is the other message type, used for constructs the parser
knows it is skipping.

## Lookahead by speculative parse

Some Delphi constructs cannot be decided from one token. The classic cases are
telling a labelled statement from an assignment, telling `TFoo<T>` from a
less-than comparison, and telling a constant declaration from a typed one.

The parser handles these by keeping a **second parser instance**:

```pascal
procedure TmwSimplePasPar.InitAhead;
begin
  if AheadParse = nil then
    AheadParse := TmwSimplePasPar.Create;
  AheadParse.Lexer.InitFrom(Lexer);
end;
```

Then, for example:

```pascal
InitAhead;
AheadParse.NextToken;
AheadParse.TypeArgs;
if AheadParse.TokenId = ptGreater then
  ...it really was a generic instantiation
```

Two things to notice.

`AheadParse` is a `TmwSimplePasPar`, **not** `Self.ClassType`. The speculative
parse runs on the base class, so it builds no nodes and touches no builder
state. Speculation is invisible to the tree by construction — a descendant
cannot accidentally emit nodes from a path that was only being tried on.

The ahead parser is also not configured like the real one: no `OnMessage`
handler, and its lexer is cloned by `InitFrom`, which does copy the define
stack. Errors during speculation are therefore swallowed, which is what you
want for a guess.

## `InterfaceOnly`

Setting `InterfaceOnly := True` makes `UnitFile` and `ProgramFile` stop after
the interface section. The implementation section, initialization,
finalization and every method body are never entered.

This is a useful lever beyond speed: because bodies are never parsed, a unit
whose *bodies* are malformed still yields a complete declaration tree. It is
the cheapest mitigation for the error-recovery problem when all you need is
structure.

## `TmwSimplePasParEx`

`DelphiAST.SimpleParserEx.pas` inserts one layer between the parser and the
builder. It adds two things.

**A name-list stack.** Dotted names (`System.SysUtils`, `TFoo.Bar.Baz`) arrive
as several tokens with junk potentially in between. `BeginName`, `AddToken`
and `EndName` accumulate the pieces, and `NextToken` is overridden to feed
every token to any name currently open:

```pascal
procedure TmwSimplePasParEx.NextToken;
var
  NameList: TNameList;
begin
  if FNameListStack.Count > 0 then
    for NameList in FNameListStack.ToArray do
      if (NameList.Count > 0) and not NameList.LastItem.EndNameCalled then
        NameList.AddToken;
  inherited;
end;
```

Each recorded token keeps `TokenPos` and `TokenLen`, so `OriginalNames` can
reconstruct the exact source spelling from the buffer, while `Names` applies
`LowerCaseNames` if set.

**A string-handling hook.** `OnHandleString: TStringEvent` is called on every
string the parser is about to hand upward — identifiers, type names, dotted
names. Its intended use is interning:

```pascal
StringPool := TStringPool.Create;
Builder.OnHandleString := StringPool.StringIntern;
```

`StringPool.pas` is an open-addressed FNV-1a table that rewrites the string
variable in place to point at a shared instance. On a large project where
`Integer` and `string` appear tens of thousands of times, this is a large
allocation saving. It is entirely optional; leave it nil and nothing interns.

`TPasLexer`, in the same unit, is a deliberately narrow facade over `TmwPasLex`
exposing only `FileName`, `PosXY`, `Token` and the underlying `Lexer`. It is
what `TNodeStack` holds, so node positions can be stamped without the stack
knowing anything about lexing.
