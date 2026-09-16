# Architecture

## The idea in one sentence

A hand-written recursive-descent parser calls one virtual method per grammar
production; a descendant overrides those methods and builds a tree as a side
effect of the parse.

## License

DelphiAST as a whole is released under the **Mozilla Public License, v. 2.0**
(`LICENSE` at the repository root), copyright 2014-2020 Roman Yankovsky et al.

The two lower layers carry their own, older notice. `SimpleParser.pas`,
`SimpleParser.Lexer.pas`, `SimpleParser.Types.pas` and
`SimpleParser.Lexer.Types.pas` descend from Martin Waldenburg's `mwPasLex` /
`mwSimplePasPar`, released in November 1999, and keep their **MPL 1.1** header
naming him as the initial developer. Both licences are file-level copyleft:
modifications to a covered file stay under it, but linking DelphiAST into a
larger work of your own does not put that work under the MPL.

The Free Pascal support directory pulls in two submodules —
`Wosi/generics.collections` and `Wosi/FPC_StringBuilder` — which carry their own
terms.

## Terminology

A handful of parsing terms recur throughout these pages. None of them mean
anything exotic.

**Token.** The smallest unit the parser deals in: one identifier, one keyword,
one operator, one literal, one semicolon. `TFoo = class` is four tokens. Turning
characters into tokens is the [lexer's](lexer.md) job, and it is the only layer
that ever looks at individual characters.

**Production.** One rule of the language's grammar — a statement of what a
construct is made of. Written out, the rule for an if-statement is:

```
IfStatement  ->  'if' Expression 'then' Statement [ 'else' Statement ]
```

That is one production. Delphi's grammar is a few hundred of them, and the
parser has one method named after each: `IfStatement`, `Expression`,
`TypeDeclaration`, `FormalParameterList`, and so on. When these pages say
"the builder overrides 184 productions", it means it supplies its own version
of 184 of those methods. "Grammar rule" would do just as well; "production" is
simply the usual word for it.

**Recursive descent.** The parsing technique where each production is a
function that calls the functions for the parts it is made of — `IfStatement`
calls `Expression`, which calls `SimpleExpression`, and so on down. The chain of
active calls mirrors the nesting of the source being read, which is what makes
the [node-stack trick](#the-inheritance-roles) work.

**Lookahead.** Deciding what a construct is by peeking at tokens beyond the
current one, without consuming them. Needed where the first token is ambiguous —
`Foo<T>` and `Foo < T` start identically.

**Syntax tree.** The output: a tree whose shape mirrors the nesting of the
source. A *concrete* tree keeps everything the grammar produced. An *abstract*
one drops whatever its consumer can do without — so how abstract a tree should
be depends on who is reading it, and "abstract" on its own says very little.

A compiler's tree can discard comments, layout and parentheses: none of them
change the code it emits, and two files differing only in those ways ought to
produce the same tree. A refactoring tool's tree cannot discard them. It has to
put text back where it came from, so exact positions matter and comments are
part of its subject matter rather than noise.

DelphiAST sits between the two, and not always where a tool of the second kind
would put it. See [limitations.md](limitations.md).

For example, it drops punctuation and keywords: there is no node for `begin`,
`end`, `then` or a semicolon. Once the statements written between a `begin` and
its matching `end` are all children of one node, the `begin` and `end` become implicit.

## The layers

```
      source text (a string)
      |
      v
  +---------------------------+               +------------------------------------------------------+
  | TmwBasePasLex / TmwPasLex |               | TmwSimplePasPar                     SimpleParser.pas |
  |   SimpleParser.Lexer.pas  | parsed tokens |   recursive descent over the Delphi grammar          |
  |   characters -> tokens    | ------------->|   341 virtual methods, one per production            |
  |   resolves {$IFDEF}       |               |   builds nothing of its own - it only walks the code |
  |   expands {$I} includes   |               |   and calls a virtual method for each production     |
  |   reports comments        |               +------------------------------------------------------+
  +---------------------------+
                                                  |  inheritance: each box below
                                                  |  is a descendant class of the one above it
                                                  v
                                              +------------------------------------------------------+
                                              | TmwSimplePasParEx       DelphiAST.SimpleParserEx.pas |
                                              |   rejoins dotted names such as System.SysUtils       |
                                              |   optional hook for sharing repeated strings         |
                                              +------------------------------------------------------+
                                                  |
                                                  v
                                              +------------------------------------------------------+
                                              | TPasSyntaxTreeBuilder                  DelphiAST.pas |
                                              |   Creates a tree of TSyntaxNodes                     |
                                              |   overrides 184 of those methods                     |
                                              |   pushes and pops nodes as the parse nests,          |
                                              |   rearranges some of the result, records names       |
                                              +------------------------------------------------------+
                                                  |
                                                  v
                                                  TSyntaxNode tree   +   Comments: TObjectList<TCommentNode>
                                                  |
                                                  +-- TSyntaxTreeWriter.ToXML
                                                  +-- TSyntaxTreeWriter.ToBinary
                                                  +-- your own walker
```

The bottom three boxes are **classes, not stages**. `TmwSimplePasParEx`
descends from `TmwSimplePasPar`, and `TPasSyntaxTreeBuilder` descends from
`TmwSimplePasParEx`. 

Only two objects exist while a parse runs: the `TPasSyntaxTreeBuilder` you
create and the lexer it owns.
The horizontal arrow is the only real hand-off in the diagram - the parser pulls
one token at a time from the lexer. The vertical arrows below it are
inheritance.

The lexer and `TmwSimplePasPar` are a fork of Martin Waldenburg's `mwPasLex` /
`mwSimplePasPar` from 1999, by way of Castalia. They carry their own licence
header and their own house style, and they know nothing about syntax trees.
`TmwSimplePasParEx` and `TPasSyntaxTreeBuilder` are DelphiAST proper.

## Unit inventory

| Unit                             | Role                                                                                  |
| -------------------------------- | ------------------------------------------------------------------------------------- |
| `SimpleParser.Lexer.pas`         | The lexer. ~83 KB, mostly dispatch tables and keyword hashing.                        |
| `SimpleParser.Lexer.Types.pas`   | `TptTokenKind` (the token enum), `TTokenPoint`, `IIncludeHandler`, `IsTokenIDJunk`.   |
| `SimpleParser.pas`               | The recursive-descent parser. ~124 KB, one method per production.                     |
| `SimpleParser.Types.pas`         | `TmwParseError` and its message strings.                                              |
| `DelphiAST.SimpleParserEx.pas`   | `TPasLexer` (a narrowing facade over the lexer) and `TmwSimplePasParEx` (name lists). |
| `DelphiAST.pas`                  | `TPasSyntaxTreeBuilder`, `TNodeStack`, `ESyntaxTreeException`.                        |
| `DelphiAST.Classes.pas`          | `TSyntaxNode` and subclasses; `TExpressionTools`.                                     |
| `DelphiAST.Consts.pas`           | `TSyntaxNodeType` (136 values), `TAttributeName` (14), and their string names.        |
| `DelphiAST.Writer.pas`           | XML and binary output.                                                                |
| `DelphiAST.Serialize.Binary.pas` | The binary format.                                                                    |
| `DelphiAST.ProjectIndexer.pas`   | Multi-unit driver. See [project-indexer.md](project-indexer.md).                      |
| `StringPool.pas`                 | An optional intern table for identifier strings.                                      |

## One parse, end to end

`TPasSyntaxTreeBuilder.Run` is the public entry point:

```pascal
class function TPasSyntaxTreeBuilder.Run(const FileName: string;
  InterfaceOnly: Boolean = False; IncludeHandler: IIncludeHandler = nil;
  OnHandleString: TStringEvent = nil): TSyntaxNode; static;
```

What happens inside:

1. The file is read into a `SourceStream: TStringStream`. There is no incremental or
   streaming mode; the whole source becomes a single `PChar` buffer.
2. `InitDefinesDefinedByCompiler` seeds the define table — see the warning in
   [lexer.md](lexer.md#conditional-compilation). The define table is used to parse code based on whether or not particular compiler defines are active
3. `TPasSyntaxTreeBuilder.Run(SourceStream)` — the instance method, not the
   class method above — does four things, each belonging to a different class
   in the chain:
   - creates the root node, a `TSyntaxNode` (`DelphiAST.Classes.pas`) of type
     `ntUnit`;
   - pushes it on `FStack`, a `TNodeStack` — `TPasSyntaxTreeBuilder`'s own
     field, declared alongside it in `DelphiAST.pas`;
   - **re-assigns `OnMessage`** — a property it inherits from
     `TmwSimplePasPar` — to its own private `TPasSyntaxTreeBuilder.ParserMessage`
     (`DelphiAST.pas:2169`);
   - calls `inherited Run`, which resolves to `TmwSimplePasPar.Run`, because
     `TmwSimplePasParEx` does not override it.
4. `TmwSimplePasPar.Run` takes the text out of that same `SourceStream` —
   `GetDataString`,  — and assigns it to `FLexer.Origin`  (Assigning `FLexer.Origin` is what allocates the `PChar`
   buffer and lexes the first token, both as side effects of the setter). `Run`
   then calls `TmwSimplePasPar.ParseFile`, which dispatches on the first keyword
   to `UnitFile`, `ProgramFile`, `LibraryFile` or `PackageFile` — all of them
   `TmwSimplePasPar` methods, overridden further down the chain.
5. Recursive descent proceeds. Every production the `TPasSyntaxTreeBuilder` cares about is
   overridden; each override typically pushes a node, calls `inherited`, and
   pops.
6. The stack unwinds to the root, `Assert(FStack.Count = 0)` fires in debug
   builds, and the root is returned.

On the way out, any `EParserException` or `ESyntaxError` is caught and
re-raised as `ESyntaxTreeException`, **carrying the partial tree** in its
`SyntaxTree` property. That is the only error recovery in the library, and it
has sharp edges — see [limitations.md](limitations.md#error-recovery).

## The inheritance Roles

This is the part worth internalising. `TmwSimplePasPar` — the parser — does not emit events, build a parse tree, or call a
visitor. It calls virtual methods on itself, and a descendant constructs a tree by overriding the methods.

Take two lines of Delphi:

```pascal
if Count > 0 then
  WriteLn(Count);
```

Working through this, the parser will draw 10 tokens from the lexer:

```
if   Count   >   0   then   WriteLn   (   Count   )   ;
```

which are represented by 10 *values of an enumeration*. The parser matches on those
enumerations, never on text:

| Token text | `TptTokenKind`   |
| ---------- | ---------------- |
| `if`       | `ptIf`           |
| `Count`    | `ptIdentifier`   |
| `>`        | `ptGreater`      |
| `0`        | `ptIntegerConst` |
| `then`     | `ptThen`         |
| `WriteLn`  | `ptIdentifier`   |
| `(`        | `ptRoundOpen`    |
| `Count`    | `ptIdentifier`   |
| `)`        | `ptRoundClose`   |
| `;`        | `ptSemiColon`    |

The lexer does the conversion in `TmwBasePasLex.Next`, which dispatches on the
first character to a handler. For a letter that handler is `IdentProc`, which
hashes the identifier and looks the hash up in a table of 192 functions; the
one it lands on does the final string comparison and returns `ptIf`, `ptThen`
or whatever keyword matched — or `ptIdentifier` if the word is not a keyword at
all. That comparison is the last time anyone looks at the characters. See
[lexer.md](lexer.md#character-dispatch).

Two consequences visible in the table. `Count` and `WriteLn` are both just
`ptIdentifier` — the lexer does not care what they name, and the spelling has
to be fetched separately through `Lexer.Token`, which cuts it out of the source
buffer on demand. And the spaces and the line break produced tokens too, but
they are [junk](lexer.md#junk-and-why-the-parser-never-sees-whitespace) and get skipped, which is why the parser sees exactly these ten.

### There is no list of tokens

The ten tokens above are what the lexer *will* produce, not a list it hands
over. Nothing is tokenised in advance and nothing holds a token array.

`TmwBasePasLex` owns the source buffer and a cursor into it, and a single
"current token" described by a handful of its own fields — `TokenID`,
`TokenPos`, `TokenLen`. Calling `TmwBasePasLex.Next` scans one token forward from the cursor
and **overwrites those fields**. The previous token is gone; there is only ever
one, the current one.

So the parser does not receive tokens, it *pulls* them, one at a time, as its
productions decide they need the next one:

```pascal
procedure TmwSimplePasPar.NextToken;
begin
  FLexer.NextNoJunk;      // Next, repeatedly, until the token is not junk
end;
```

## What iterates, then?

There is no loop over tokens anywhere — advancing through tokens is spread
across the productions themselves, and almost all of it happens inside the method
`Expected`, which consumes exactly one token per call. The parser gets from the
first token to the last simply because every production consumes the tokens it needs
and returns.

Where a construct genuinely repeats, the loop is local to the production for it.
Statements are the example.

A **statement** is one executable step in a method body — something the code
*does*. An assignment `X := 1`, a call `WriteLn(Count)`, an `if`, a `while`, a
`try`, or a whole `begin..end` group  (which counts as a single statement that
happens to contain others). Statements are what sits between `begin` and `end`,
separated by semicolons. Two things they are not:

- not **declarations** — `type`, `const`, `var`, a procedure heading — which
  say what something *is* rather than what to do, and appear *before* the
  `begin`. That is why the parser reaches them through
  `DeclarationSections` and statements through `Statements`;
- not **expressions**, which compute a value instead of performing an action.
  `Count > 0` is an expression; `if Count > 0 then WriteLn(Count);` is a
  statement that contains an expression.

This is the loop that will eventually reach our `if`:

```pascal
procedure TmwSimplePasPar.Statements;
begin
  while TokenID in [ptAsm, ptBegin, ptCase, ptFor, ptIdentifier, ptIf,
                    ptRepeat, ptTry, ptWhile, ptWith, ...] do
  begin
    Statement;
    Semicolon;
  end;
end;
```

Each turn parses one statement and its terminating semicolon, and the loop
ends when the current token is not something a statement can start with —
`end`, typically. `DeclarationSections` and `ImplementationSection` have the
same shape one level up, looping while the current token is `type`, `const`,
`procedure` and so on.

### How `IfStatement` gets called

Following it from the top, every step an ordinary method call:

```
TPasSyntaxTreeBuilder.Run
  -> TmwSimplePasPar.Run              assigns the buffer, lexes the first token
  -> ParseFile                        sees 'unit' -> UnitFile
  -> UnitFile
  -> ImplementationSection            loops over declarations
  -> DeclarationSection ...
  -> ProcedureDeclarationSection      a procedure's heading
  -> Block                            its declarations, then its body
  -> CompoundStatement                Expected(ptBegin), Statements, Expected(ptEnd)
  -> Statements                       the while loop above
  -> Statement                        one statement - and our token is 'if'
```

`TmwSimplePasPar.Statement` is the production for "one statement". It looks at
the current token and dispatches on it:

```pascal
procedure TmwSimplePasPar.Statement;
begin
  case TokenID of
    ptAsm:   AsmStatement;
    ptBegin: CompoundStatement;
    ptCase:  CaseStatement;
    ptFor:   ForStatement;
    ptIf:    IfStatement;        // <- our token is 'if', so we go here
    ...
```

`IfStatement` is then the grammar rule, transcribed line for line:

```pascal
procedure TmwSimplePasPar.IfStatement;
begin
  Expected(ptIf);                // the token must be 'if' - consume it
  Expression;                    // parse the condition
  ThenStatement;                 // consume 'then', then parse one statement
  if TokenID = ptElse then
    ElseStatement;
end;
```

### What each call does to the token stream

Every production consumes its own span of tokens and leaves the current token
sitting on whatever comes next. That is the contract the whole parser relies on:

| Call             | Tokens it consumes       | Current token when it returns |
| ---------------- | ------------------------ | ----------------------------- |
| `Expected(ptIf)` | `if`                     | `Count`                       |
| `Expression`     | `Count > 0`              | `then`                        |
| `ThenStatement`  | `then WriteLn ( Count )` | `;`                           |

`Expected(Sym)` is the workhorse, and it does two jobs at once: it checks the
current token against `Sym`, and **on a match calls `NextToken` to advance**.
On a mismatch it reports an error through `OnMessage` and does *not* advance.
So `Expected(ptIf)` reads as "the next token had better be `if`; consume it".

`Expression` parses the condition and stops at the first token that cannot
continue an expression — here, `then`.

`ThenStatement` is where the recursion becomes visible:

```pascal
procedure TmwSimplePasPar.ThenStatement;
begin
  Expected(ptThen);
  Statement;          // <- the same production we started in
end;
```

It consumes `then` and parses one more statement. Nesting in the source becomes
nesting in the call stack: `if A then if B then C` runs
`Statement` → `IfStatement` → `ThenStatement` → `Statement` → `IfStatement`,
four levels deep and still only ordinary method calls.

Note what is *not* happening: none of this produces anything. Run
`TmwSimplePasPar` over that fragment and the only trace left behind is how far
the lexer moved.

### Where the tree comes from

The builder overrides the same method:

```pascal
procedure TPasSyntaxTreeBuilder.IfStatement;
begin
  FStack.Push(ntIf);
  try
    inherited;        // runs the version above: Expected, Expression, ThenStatement
  finally
    FStack.Pop;
  end;
end;
```

`IfStatement` is `virtual`, and the object doing the parsing is really a `TPasSyntaxTreeBuilder`,
so the call made from `Statement` arrives *here* first — even though `Statement`
was written in `TmwSimplePasPar` and knows nothing about trees. `inherited` then
runs the original, and everything it parses happens while the `ntIf` node is on
top of the stack.

That is the whole trick. Every nested override adds to `FStack.Peek`, so the
condition, the `then` branch and anything nested inside them all land under that
`ntIf` node. The node stack *is* the parser's call stack, mirrored in data.

Three consequences:

- **The tree shape is the grammar's shape**, except where the builder
  explicitly rearranges it. See are there[tree-builder.md](tree-builder.md#where-the-tree-is-reshaped) for the places
  that happens.
- **Extension means overriding more productions.** There are 341 virtuals and
  184 are already taken; the rest are free hooks. See
  [extending.md](extending.md).
- **There is no separate AST pass.** Nothing walks the tree after the parse to
  fix it up, so anything the parse could not see is simply absent — most
  visibly, the inactive branches of `{$IFDEF}`.

## Ownership and lifetime

- `TSyntaxNode.Destroy` frees its children recursively. The root owns the whole
  tree; free the root and you are done.
- Nodes hold no reference back to the builder or the lexer, and `FileName` is
  copied per node, so the tree outlives the builder safely.
- **Comments are not in the tree.** They accumulate in
  `TPasSyntaxTreeBuilder.Comments`, a `TObjectList<TCommentNode>` owned by the
  builder, and are destroyed with it. If you want them, take them before the
  builder goes out of scope — and note that the class-method `Run` frees the
  builder for you, so comments are unreachable through that route entirely.
- `ESyntaxTreeException` owns the partial tree it carries and frees it in its
  destructor. Set `e.SyntaxTree := nil` to claim it.

## What this design is not

- **Not a compiler front end.** No symbol table, no type resolution, no
  overload selection. An identifier node is a name and a position, nothing more.
- **Not project-aware.** One file at a time. `TProjectIndexer` is a driver on
  top, not a deeper analysis.
- **Not round-trippable.** There is no Pascal emitter, and the tree does not
  retain enough to be one. See
  [limitations.md](limitations.md#the-tree-cannot-be-printed-back-to-source).
- **Not incremental.** Every parse is from scratch. It is fast enough that this
  rarely matters.
