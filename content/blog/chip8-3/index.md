+++
draft=true
title = "Syntax checking complete-ish"
date = "2025-09-16"
description = "Building a Chip-8 IDE in Compose Multiplatform: tokenization, parsing, and real progress."
tags = [
    "emulation",
    "kotlin",
    "compose",
    "programming",
]
+++

Progress update! Nachos—the Chip‑8 IDE I’m building—now has a feature‑complete tokenizer and syntax checker for the Octo language. It successfully parses every Octo example I’ve thrown at it except one. I’ll share that tiny exception later, but first, a quick tour of how the tokenizer and parser work, what they catch, and what I learned along the way.

## Tokenization: turning text into tokens

Tokenization is the first step in compilation. The source text becomes a stream of tokens—objects that carry a type (Identifier, Number, Plus), position (line, column), and sometimes a value (identifier name, numeric literal, string contents).

For example, this Octo snippet:


```octo

Our first do-nothing function
: noop
return
```
becomes this list of token objects:


```kotlin
Colon(line=0, column=0)
Identifier(name="noop", line=0, column=2)
Return(line=1, column=0)
```

The tokenizer:
- strips comments and whitespace,
- recognizes identifiers, numbers, and strings,
- identifies symbols/operators like \+, \-, \:, \&lt;&lt;, \:=,
- tags keywords like Return, Scroll\-Left, Exit, etc.

Tokenization does not validate logic or structure. It doesn’t know if you forgot a parameter or mismatched parentheses—it just recognizes the pieces.

### Tokenizer shape (pseudocode)

A tokenizer typically reads characters, decides what token kind could start at that position, consumes the rest, and emits a token:


```kotlin
// Pseudocode
while (canContinue()) {
    val ch = peekChar()
    when {
        ch == '#' -> consumeComment()
        ch.isWhitespace() -> consumeWhitespace()
        ch.isLetter() || ch == '_' || ch == ':' -> consumeIdentifierOrLabelOrDirective()
        ch.isDigit() -> consumeNumber() // e.g., 0x.., 0b.., decimal
        ch == '"' -> consumeStringOrError()
        else -> consumeSymbolOrError() // operators, punctuation, or unknown
    }
}
```
Some errors can be detected here. For instance, Nachos emits an Error token if it sees an opening quote without a matching closing quote:


``kotlin
// Example outcome
String(line=10, column=15, value="incomplete
Error(line=10, column=15, message="Unterminated string literal")
``

## Parsing: building structure and catching mistakes

Once we have tokens, the parser turns them into higher‑level constructs and performs syntax checks. Octo is a small assembly-like language:
- arithmetic groups with parentheses and evaluates left‑to‑right,
- types are limited (numbers, identifiers, strings),
- macros are simple,
- control flow is jumps/ifs/loops,
- registers are manual.

This keeps the parse tree fairly linear and the parser straightforward. Unlike tokenization, parsing enforces rules and evaluates context:
- expands and validates macros,
- verifies identifiers are defined,
- matches braces/parentheses,
- checks expected types and arities,
- annotates nodes with metadata for the assembler.

The result is a list of ParseTokens. When something goes wrong, the parser emits Error nodes with clear, positional messages—but keeps going to surface as many issues as possible in one pass.

### Example: macro expansion

Input:


```octo
:macro foo SIZE {
v0 := SIZE
i := CALLS # CALLS is the number of times this macro has been expanded.
}
foo 0x42
foo 0x42
```
Parsed summary:


```text
Macro, tokens size: 11, lines [1, 2, 3, 4]
MacroExpand, tokens size: 2, lines [5]
Assignment, tokens size: 3, lines [2, 5]
IAssign, tokens size: 3, lines [3, 0]
MacroExpand, tokens size: 2, lines [6]
Assignment, tokens size: 3, lines [2, 6]
IAssign, tokens size: 3, lines [3, 0]
```

Zooming into one expansion to show parameters and substitutions:


```text
MacroExpand, tokens size: 2, lines [5]
Identifier(name=foo, line=5, column=12) Number "66" 5:16
Assignment, tokens size: 3, lines [2, 5]
Register(register=v0, line=2, column=16) Assignment(line=2, column=19) Number "66" 5:16
IAssign, tokens size: 3, lines [3, 0]
Register(register=i, line=3, column=16) Assignment(line=3, column=18) Number "1" 0:0
```

### Example: catching errors (and continuing)

If we pass the wrong type and then forget an argument:


```octo
:macro foo SIZE {
    v0 := SIZE
    i := CALLS
}
foo "hello"
foo
```

The parser emits errors but continues analysis so the user gets multiple helpful messages in one run:


```text
MacroExpand, tokens size: 2, lines [5]
Identifier(name=foo, line=5, column=0) String "hello" 5:4
Error, tokens size: 4, lines [2, 5]
Register(register=v0, line=2, column=4) Assignment(line=2, column=7) String "hello" 5:4 Error "Expected Register, Identifier, Number or Key" 5:4
IAssign, tokens size: 3, lines [3, 0]
Register(register=i, line=3, column=4) Assignment(line=3, column=6) Number "1" 0:0
Error, tokens size: 3, lines [6]
Identifier(name=foo, line=6, column=0) Error "Unexpected end of program" 6:0 Error "Error parsing macro foo" 6:0
```

The first macro expansion fails because a String can’t be assigned to a register. The second fails because the invocation is incomplete.

## The one exception

One example file didn’t parse on the first try: [caveexplorer.8o, line 222](https://github.com/JohnEarnest/Octo/blob/76f816c8e36891478e1caf5209d85a95d16a38dc/examples/caveexplorer.8o#L222). A label happens to use the name exit, which is a SuperChip‑8 keyword. It’s a fun edge case—and also a nice sign that the project is far enough along to surface real, actionable issues from real programs. Real code is already producing real feedback.

## Lessons learned

- Fail fast at the character level, fail friendly at the syntax level.
- Keep tokens rich: line/column and literal values make downstream errors precise.
- Make macros first‑class in the parser: preserve context for better diagnostics and assembly.
- Small language, simple parser: lean into Octo’s linearity and constraints.
- Real code > synthetic tests: example suites flush out keyword/name collisions and corner cases.

## What’s next

- Wire the tokenizer and parser into the editor for live diagnostics.
- Build the assembler so Octo programs can run end‑to‑end.
- Add debugging tools in the IDE: state inspection, breakpoints, step/run, and register/memory views.

Still plenty to do, but Nachos is now producing real value—and that’s delicious.

