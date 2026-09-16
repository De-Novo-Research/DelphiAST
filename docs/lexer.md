# The lexer

`SimpleParser.Lexer.pas`, classes `TmwBasePasLex` and `TmwPasLex`.

This is 1999 code optimised for its era: no regular expressions, no state
machine tables generated from a grammar, just two hand-built dispatch arrays
and a lot of pointer arithmetic over a single `PChar`. It is also the only
layer that knows about conditional compilation and include files, which is why
several surprising behaviours further up the stack originate here.

## Token representation

A token is not an object. After each `Next`, the lexer's own fields describe
the current token:

| Member | Meaning |
|---|---|
| `TokenID: TptTokenKind` | The token class. 250-odd values in `SimpleParser.Lexer.Types.pas`. |
| `ExID: TptTokenKind` | The *extended* id. For a `ptIdentifier` that happens to be a context-sensitive word (`read`, `write`, `name`, `index`, `operator`, ...) this holds that word's kind; otherwise `ptUnknown`. |
| `GenID` | `ExID` if set, else `TokenID`. The parser mostly branches on this. |
| `Token: string` | The token text, cut from the buffer on demand. |
| `TokenPos`, `TokenLen` | Offset and length in the buffer — the only way to get an exact character range. |
| `PosXY: TTokenPoint` | `X` = column, `Y` = line, both 1-based. |

`ExID` is how Pascal's context-sensitive keywords are handled without a
symbol table: `read` is an identifier everywhere except where the parser
chooses to look at `ExID`.

## Character dispatch

`MakeMethodTables` fills `FProcTable: array[#0..#127] of procedure of object`,
one entry per ASCII character, at construction. `Next` is then:

```pascal
procedure TmwBasePasLex.Next;
begin
  FExID := ptUnKnown;
  FTokenPos := FBuffer.Run;
  FTokenLine := FBuffer.LineNumber;
  FTokenLinePos := FBuffer.LinePos;
  case FCommentState of
    csNo: DoProcTable(FBuffer.Buf[FBuffer.Run]);
    csBor: BorProc;
    csAnsi: AnsiProc;
  end;
end;
```

One indexed indirect call per token. Anything above `#127` falls through to
`IdentProc`, so non-ASCII identifier characters are accepted without a table
entry.

Keywords are recognised by a second table. `KeyHash` walks the identifier
computing a hash bounded to 192 buckets; `IdentKind` indexes
`FIdentFuncTable: array[0..191] of function: TptTokenKind of object` and calls
the bucket's function, which does the final string comparisons for the handful
of keywords that collide there. Buckets with no keyword are bound to
`AltFunc`, which returns `ptIdentifier`.

## Junk, and why the parser never sees whitespace

```pascal
function TmwBasePasLex.GetIsJunk: Boolean;
begin
  Result := IsTokenIDJunk(FTokenID)
            or (FUseDefines and (FDefineStack > 0) and (TokenID <> ptNull));
end;
```

`IsTokenIDJunk` covers whitespace, line breaks, all three comment forms, and
every conditional-compilation directive token. The parser's `NextToken` calls
`Lexer.NextNoJunk`, which loops `Next` until the token is not junk.

Note the second half of that expression. **Tokens inside an inactive
conditional branch are junk.** This is the entire implementation of
conditional compilation as far as the parser is concerned: not "skip a
region", but "every token in the region is whitespace".

## Conditional compilation

`FDefineStack: Integer` counts how deep inside *inactive* text the lexer is.
`FTopDefineRec: PDefineRec` is a linked stack of one record per open
conditional, recording whether the branch was taken and what `FDefineStack`
was on entry.

```pascal
procedure TmwBasePasLex.EnterDefineBlock(ADefined: Boolean);
begin
  New(StackFrame);
  StackFrame^.Next := FTopDefineRec;
  StackFrame^.Defined := ADefined;
  StackFrame^.StartCount := FDefineStack;
  FTopDefineRec := StackFrame;
  if not ADefined then
    Inc(FDefineStack);
end;
```

`{$DEFINE}` and `{$UNDEF}` mutate the define list only when
`FDefineStack = 0`, so a define inside dead code is correctly ignored.

Three things about this are worth knowing before you rely on it.

**The default defines are your compiler's, not the target's.**
`InitDefinesDefinedByCompiler` is a wall of `{$IFDEF VER360} AddDefine('VER360')
{$ENDIF}`, evaluated when *your* binary was compiled. Parsing a unit written
for a different Delphi version gives you your version's view of it. Push the
target's defines in yourself with `AddDefine` (and consider
`ClearDefines` first).

**`{$IF}` understands very little.** `EvaluateConditionalExpression` handles
exactly two shapes: a comparison of `CompilerVersion` or `RTLVersion` against
an integer literal, and a flat chain of `Defined(X)` / `not Defined(X)` joined
by `and` / `or`. There is a `{ TODO }` in the source saying as much. Anything
else — parentheses, `Declared()`, arithmetic, a defined constant — evaluates
to **False silently**, taking the `{$ELSE}` branch with no diagnostic.

**`{$IFOPT}` is always false.** `EnterDefineBlock(False)` unconditionally.

Consequences for the tree are in
[limitations.md](limitations.md#conditional-compilation-is-resolved-not-recorded).

## Include files

`{$I foo.inc}` is handled by pushing a new buffer:

```pascal
TBufferRec = record
  Buf: PChar;
  Run: Integer;
  SharedBuffer: Boolean;
  LineNumber: Integer;
  LinePos: Integer;
  FileName: string;
  Next: PBufferRec;     // the buffer we came from
end;
```

`IncludeFile` asks the `IIncludeHandler` for the content, allocates a buffer,
links it to the current one and switches. `NullProc` at end-of-buffer pops back.
Because `FileName` is per buffer and `PosXY` is computed from the current
buffer's counters, **positions inside an include file are relative to that
file**, and every node records which file it came from.

Supplying an `IIncludeHandler` is optional; without one, `{$I}` is skipped
silently and whatever the include contained is simply missing from the tree.
Include expansion is also gated on `FDefineStack = 0`, so includes in dead
branches are not read.

## The ahead lexer

`TmwPasLex` adds `FAheadLex: TmwBasePasLex` and `InitFrom`, which clones the
run position, token state and the *whole define stack* into a second lexer.
`AheadToken` / `AheadTokenID` / `AheadExID` let the parser look one or more
tokens forward without disturbing the real position. The parser layer builds on
this with a second parser instance — see
[parser.md](parser.md#lookahead-by-speculative-parse).

## Multi-line strings

`StringProc` counts opening quotes and, on three or more, switches to Delphi
12's multi-line string literal rules. `StringContent` strips the quoting. This
is also where `FLineSeq` is maintained — see below.

## A defect: `TTokenPoint.LineSeq` is never set

`TTokenPoint` has three fields, `X`, `Y` and `LineSeq`. The lexer maintains an
`FLineSeq` counter, incrementing it on every line break including those inside
multi-line strings and comments. But the current `GetPosXY` does not copy it
out:

```pascal
function TmwBasePasLex.GetPosXY: TTokenPoint;
begin
  Result.Y := FTokenLine + 1;
  Result.X := FTokenPos - FTokenLinePos + 1;
end;
```

`TTokenPoint` is an unmanaged record, so `Result.LineSeq` is whatever was on
the stack. The assignment existed in `7cdb776` and again in `ba42d25`; it was
lost in the merge `42a5599`, and upstream `master` is missing it today.

Everything downstream inherits the garbage: `AssignLexerPositionToNode` copies
`PosXY.LineSeq` into every node, and `TSyntaxTreeWriter` emits it as
`line_seq="..."` on every XML element. Parsing a twelve-line unit and printing
each node's `Line` beside its `LineSeq` gives:

```
unit             line=1   col=1   lineSeq=6223848
  interface        line=2   col=1   lineSeq=6223808
    typesection    line=3   col=1   lineSeq=6223760
      typedecl     line=4   col=3   lineSeq=6223708
```

Do not use `LineSeq` or `line_seq`. Use `Line`.
