+++
drfat=true
title = "Syntax checking complete-ish"
date = "2025-09-9"
description = "I'm building an IDE for Chip 8 using Compose Multiplatform."
tags = [
    "emulation",
    "kotlin",
    "compose",
    "programming",
]
+++

Progress update folks, Nachos' syntax checker and tokenizer for the Octo language compiler is feature complete, and it parses all but one of the files I've thrown at it from the [Octo example](https://github.com/JohnEarnest/Octo/tree/gh-pages/examples). I'll talk about that one exception in a bit, but first I want to talk about how the tokenizer and parser work and lessons learned.

Tokenization is the first step of compilation. During this step, the text source file is turned into basic compnents called tokens. In Nachos, a token includes the line and column the token is found at and the type of the token. Some types of tokens contain more information. The identifier type includes a name, and the number type includes a numeric value, etc.  

So in practice, tokenization turns this text file

```octo
# Our first do nothing function
: noop
return
```

Into a list of token objects

```kotlin
Colon(line=0, column=0)
Identifier(name="noop", line=0, column=2)
Return(line=1, column=0)
```

The tokenization process removes comments and whitespace; does some basic typing for identifiers, strings, and numbers; identifies symbols and operators like Plus, Minus, Colon, Shift, Assign, etc.; and identifies keywords like "Return", "Scroll-Left", "Exit", etc.  Tokenization doesn't check for logic errors, typos, missing parameters, uneven braces or parenthesis, etc., it only identifies and lists tokens in a file.  More complex analysis is done during parsing/syntax checking.

Writing a tokenizer is pretty simple. The tokenizer reads in the sourcefile one character at a time, checks what types of tokens can be started with that character, and reads in more until it can determine a token type. Once the token type is determined, then the rest of the token's string value is consumed until some delimiter is reached. Then the token's string can be turned into a token object when one is complete. In code this looks like this:

```kotlin
//Warning, pseudocode
//The exact characters you match on depend on your language

while (canContinue()) {
        val character = getCharacter()
        if (character == '#') { //Start comment
            consumeComment()
        } else if (character.isWhitespace()) { //consume whitespace
            consumeWhitespace()
        } else if (character.isLetter()) { // consume identifier
            consumeIdentifierOrDirectiveOrRegister()
        } else if (character.isNumber()) {
            consumeNumber()
        }
        //etc
}


```

There are some errors that can be detected during tokenization. In Nachos I created a "String" token. Strings are surrounded by quotes. When the tokenizer encounters a '"' character, it will begin taking characters until it encounters a closing '"'. If it reaches the end of the line or the end of the file and does not encounter a closing quote, it will return an "Error" type token instead of a "String".

Once the compiler has a string of tokens, it can begin parsing them into more meaningful structures.  Octo is a very simple assembly language.  Arithmitic expressions are grouped by parenthesis and evaluated left to right instead of following an order of operations, there are no user-defined functions, the types are limited to numbers; identifiers; and strings, macros and string functions are very simple, there is no memory management, control functions are limited to jumps; ifs; and loops, registers are manually allocated, etc. These factors lead to the parse tree being very linear, and the parser being very straightforward.

Similar the tokenizaer consuming one character at a time, the parser consumers on token at a time and matches it to the available parse types.  Unlike tokenizaiton, the parser does syntax checking and some evaluations: macros get expanded and verified, identifiers are confirmed to be defined,  braces and parenthesis matching is enforced, expected types are checked, etc. The Nachos parser will return a listed of ParseTokens once parsing is done. These Parse tokens include metadata to be consumed by the assembler and error information if there was a problem.

The following program :

```octo
:macro foo SIZE {
    v0 := SIZE
    i := CALLS # CALLS is a built-in parameter for the number of times the macro was called.
}
foo 0x42
foo 0x42
```

becomes the parsed list

```
Macro, tokens size : 11, lines [1, 2, 3, 4]
MacroExpand, tokens size : 2, lines [5]
Assignment, tokens size : 3, lines [2, 5]
IAssign, tokens size : 3, lines [3, 0]
MacroExpand, tokens size : 2, lines [6]
Assignment, tokens size : 3, lines [2, 6]
IAssign, tokens size : 3, lines [3, 0]
```

As you can see, after each "MacoExpand" parse token, the macro contents are added to the output. This is because we've expanded the macro and inserted its contents into the parsed list.  We can dig into our output more to verify that the assignments are expanded using the correct parameters from the macro.

```
MacroExpand, tokens size : 2, lines [5]
	Identifier(name=foo, line=5, column=12) Number "66" 5:16
Assignment, tokens size : 3, lines [2, 5]
	Register(register=v0, line=2, column=16) Assignment(line=2, column=19) Number "66" 5:16
IAssign, tokens size : 3, lines [3, 0]
	Register(register=i, line=3, column=16) Assignment(line=3, column=18) Number "1" 0:0
MacroExpand, tokens size : 2, lines [6]
	Identifier(name=foo, line=6, column=12) Number "66" 6:16
Assignment, tokens size : 3, lines [2, 6]
	Register(register=v0, line=2, column=16) Assignment(line=2, column=19) Number "66" 6:16
IAssign, tokens size : 3, lines [3, 0]
	Register(register=i, line=3, column=16) Assignment(line=3, column=18) Number "2" 0:0

```

If we introduce an error the syntax checker will pick it up.  Let's use a wrong type and forget a parameter.

```octo
:macro foo SIZE {
    v0 := SIZE
    i := CALLS # CALLS is a built-in parameter for the number of times the macro was called.
}
foo "hello"
foo 
```

Now we have "Error" parse tokens in our output stream. This will signal the assembler to stop, and it will be reported to the user. One feature is that even though there was an error, the assembler still tries to keep going. 

```
MacroExpand, tokens size : 2, lines [5]
	Identifier(name=foo, line=5, column=0) String "hello" 5:4
Error, tokens size : 4, lines [2, 5]
	Register(register=v0, line=2, column=4) Assignment(line=2, column=7) String "hello" 5:4 Error "Expected Register, Identifier, Number or Key" 5:4
IAssign, tokens size : 3, lines [3, 0]
	Register(register=i, line=3, column=4) Assignment(line=3, column=6) Number "1" 0:0
Error, tokens size : 3, lines [6]
	Identifier(name=foo, line=6, column=0) Error "Unexpected end of program" 6:0 Error "Error parsing macro foo" 6:0
```

Out output confirms that the first macro expansion fails because strings can't be assigned to a register, and the second expansion fails because the file ends while it is still expecting more tokens.

So what's next for Nachos? First, I'm going to integrate the tokenizer and syntax output into the editor. Then I'm going to implement the assembler so text programs can be run. Finally, I will implement debugging tools into the IDE. It is still a lot of work, but it should be educational.