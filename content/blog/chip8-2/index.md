+++
title = "Chip8 IDE with Compose - A TDD Assembler"
date = "2025-08-12"
description = "I'm building an IDE for Chip 8 using Compose Multiplatform."
tags = [
    "emulation",
    "kotlin",
    "compose",
    "programming",
]
+++

 I like Test-Driven Development (TDD) because it helps me overcome the hurdle of starting from no code when beginning a project and focus on breaking down problems and writing code. Reading **Test Driven Development: By Example** by Kent Beck was a turning point for me. I also practiced with the **Test-First Challenge** by Bill Wake. Between these two resources, I truly embraced the practice, and it has been immensely helpful. TDD encourages iterative, incremental steps to build a project and provides reassurance that refactors haven't broken working code. TDD feels like a natural fit for writing  my Chip8 IDE [Nachos](https://github.com/secondsun/nachos), at least for the compiler component.

I probably won't be writing many tests for the UI, but I am following TDD for the compiler[^1].  I've done this a couple of times before for my [SuperFX](https://github.com/secondsun/retro-lsp) and [WLA](https://github.com/secondsun/wla4j) language servers, and it is a great way to start. It makes me learn the languages I'm implementing at a deeper level, and the tests create APIs that are easy to use and extend by other tools. In fact, having my tests be the first "user" of my code can create really useful APIs, though the code will probably be badly encapsulated (that's what refactoring is for).  

I'm following the [Octo Assembly Language](https://github.com/JohnEarnest/Octo/blob/gh-pages/docs/Manual.md) manual and writing tests as it says things[^2]. So far I've started the first paragraph:

> Octo programs are a series of `tokens` separated by `whitespace`. Some tokens represent Chip-8 `instructions` and some tokens are `directives` which instruct Octo to do some special action as the program is compiled. The `:` directive, followed by a name (which cannot contain spaces) defines a `label`. A label represents a `memory address`- a location in your program. **You must define at least one label called main** which serves as the entrypoint to your program.

These are a lot of things! We have whitespace separating tokens, directive tokens, labels, and memory addresses as well as a requirement for a main label. I wrote the following two tests which capture much of this.

```kotlin
   @Test
   fun programMustContainMain() {
       // Initialize the Assembler
       val assembler = Assembler()

        //First, we verify that the absence of a main label causes the program to fail
       val programWithoutMain = """
           : pain
              clear
              jump pain
       """.trimIndent()

       var program = assembler.compile(programWithoutMain)

       assertTrue(program.hasErrors(), "Program with out main should have errors")

       //Next, we verify that the presence of a main label causes the program to pass.
       val programWithMain = """
           : main
              clear
              jump main
       """.trimIndent()

       var program = assembler.compile(programWithMain)
       assertFalse(program.hasErrors(), "Program with main should not have errors")
   }
```

When I wrote this test I just assumed I had things working the way I wanted. I want the Assembler to return a program, and the program should be an object that lets me inspect the state of the assembly. This is almost certainly not my final design; Assembler should probably be a functional object instead of an class, "compile" should probably be named "assemble", and the program should be a linkable result object that is then compiled into a Chip8 binary. However, none of those things are my problem right now. Right now my problem is the tests don't compile.

To make the tests compile I've created all of my classes and stubbed out methods.

```kotlin
class Assembler {
    fun compile(program: String): Program {
        return Program()
    }
}

class Program() {

    val errors = mutableListOf<Error>()

    fun addError(error: Error): Program{
        errors.add(error)
        return this
    }

    fun hasErrors(): Boolean {
        return errors.isNotEmpty()
    }

}

sealed interface Error {
    object NoMain : Error
}

```

The test compiles and runs![^3] Unfortunately, the test also fails; how do I get it to pass[^4]? It is completely valid in TDD to do the bare minimum to get it to pass as each test informs the next step in design. Starting with failing tests and incrementally adding just enough functionality to pass them keeps the development process focused and lean. Since I want to ensure there is a `main` label, I could write the following: 

```kotlin
fun compile(program: String) : Program {
    return Program().apply {
        if (!program.contains(": main")) {
            addError(NoMain)
        }
    }
}

``` 

and the test would pass[^5]. Instead, I've chosen to actually write a proper tokenizer (and tests for it) and leave this test failing for now. Failing tests serve as excellent markers to remember where you left off, especially for hobby projects. I know that when I come back I can just build the project and get a giant red label pointing me to where I wanted to start when I stopped working.

Speaking of leaving a hobby project in the middle of something...



# Links
* [Test Driven Development: By Example](https://www.amazon.com/Test-Driven-Development-Kent-Beck-ebook/dp/B095SQ9WP4)
* [Test-First Challenge by Bill Wake](https://xp123.com/test-first-challenge/)
* [Octo Assembly Language](https://github.com/JohnEarnest/Octo/blob/gh-pages/docs/Manual.md)
* [SuperFX](https://github.com/secondsun/retro-lsp) language server
* [WLA](https://github.com/secondsun/wla4j) language server
* [Nachos](https://github.com/secondsun/nachos)

[^1]: I find TDD for UI tests frustrating to write because UI is so visual, and the testing frameworks feel very tightly coupled to the UI implementation that makes refactoring a lot of work. I do write UI tests, but they tend to be smoke style tests using something like Selenium. I've also TDD'd in weird places, like my [SuperFX 3D engine](https://github.com/secondsun/snes-sfx-demo/tree/prototype/X-GSU/tests).

[^2]: "Things" is the technical term.

[^3]: Quick send it to production!

[^4]: [Requisite Invincible Meme](https://en.meming.world/wiki/That%27s_the_neat_part,_you_don%27t)

[^5]: Ok, now can we send it to production?
