## Building an Advent of Code Nom Parser for Beginners

### Or: How My Advent of Code Nom Parser works

This article is an attempt to lay out the basic workflow for a Nom beginner to parse Advent of Code puzzle input. Advent of Code is a great place to learn a new way to parse input, since the inputs can be quite strange.

To start, I'd like to introduce some ideas for how I'll talk about parsing. A [parser in Nom](https://docs.rs/nom/latest/nom/type.IResult.html) is just a Rust function
that uses the IResult type:

```Rust
fn round(input: &str) -> IResult<&str, Round>
```

And the signature of the [IResult type:](https://docs.rs/nom/latest/nom/type.IResult.html)
```Rust
pub type IResult<I, O, E = Error<I>> = Result<(I, O), Err<E>>;
```

The Ok variant of the IResult is a wrapper around a tuple formatted: (processed input, unprocessed input). We can use
the error variant of IResult to control which parser we'll use next, for example using an alt parser combinator. The
workflow, then, is to build the smallest parsers we reasonably can, and then compose these tiny parsers using parser combinators.

I recommend using the AoC example input in a test function, and using the [debug](https://doc.rust-lang.org/std/macro.dbg.html) macro to check each parser is working how you expect it to be.

Let's take the [cube game](https://adventofcode.com/2023/day/2) (AoC 2023 day 2, part 1) as an example.
The input for the puzzle looks like this:

> Game 1: 3 blue, 4 red; 1 red, 2 green, 6 blue; 2 green <br>
> Game 2: 1 blue, 2 green; 3 green, 4 blue, 1 red; 1 green, 1 blue <br>
> Game 3: 8 green, 6 blue, 20 red; 5 blue, 4 red, 13 green; 5 green, 1 red <br>
> Game 4: 1 green, 3 red, 6 blue; 3 green, 6 red; 3 green, 15 blue, 14 red <br>
> Game 5: 6 red, 1 blue, 3 green; 2 blue, 1 red, 2 green <br>

The way to approach Nom parsing is bottom-to-top. Looking at the example, it's clear the the number of cubes is going
to be important, so let's start there.

Nom has a built-in parser for parsing an integer called [u32](https://docs.rs/nom/latest/nom/character/complete/fn.u32.html).
We want to take a text number and turn it into the Rust 32-bit unsigned integer type, confusingly the [u32](https://doc.rust-lang.org/std/primitive.u32.html) type.
The u32 Nom parser will give us a u32 Rust integer.

The colors next to the numbers tell us what kind of cubes we're dealing with. I used enums with u32s inside them:

```Rust
enum Block {
    Red(u32),
    Green(u32),
    Blue(u32),
}
```

In Nom, text indicating what a number means can be thought of as a [tag](https://docs.rs/nom/latest/nom/bytes/complete/fn.tag.html).
There is also a space to deal with. Spaces being so common, there's a built-in parser called space1. So in game 1,
round 1 we have 3 blue cubes represented by "3 blue". The [tuple](https://docs.rs/nom/latest/nom/sequence/fn.tuple.html)
sequence applies each of its parsers one by one and returns their results as a tuple.

```Rust
tuple((space1, tag("blue"))))
```

When this parser matches, we know the number that came before it represents blue cubes. That means we should use the
u32 parser to build a Blue variant of the Block enum only if it ends with " blue". This we can handle with the
Nom [terminated](https://docs.rs/nom/latest/nom/sequence/fn.terminated.html) sequence combinator.

```Rust
terminated(u32, tuple((space1, tag("blue");
```

We can think of this code as its own parser for blue cubes. That is, it only
'succeeds' when we have a string that matches the form "some number followed by a space and the word 'blue'". Any other
situation, the parser returns a Nom error.

 ```Rust
fn blue(input: &str) -> IResult<&str, Block> {
let (remaining, n) = terminated(u32, tuple((space1, tag("blue"))))(input)?;
Ok((remaining, Block::Blue(n)))
}
```
Of course we don't just have blue cubes. We want to be able to say 'match red, green, or blue cubes'. This can be done
with an `alt()` combinator. It does what it says on the box: Combine parsers as alternatives. After we have written the
red and blue parsers, we'll combine them as so:

```Rust
alt((red, blue, green))
````

There's some text around each block that indicates it's a block. Let's look at the first game again.

> Game 1: 3 blue, 4 red; 1 red, 2 green, 6 blue; 2 green <br>

So the first block of the first game is 3 blue, 4 red. We know this because it's between the colon and the semicolon.
The second block is between the semicolons, and the last block is between a semicolon and a line break. The built-in
delimited parser is perfect here. If you aren't familiar with the word `delimited()`, I always think about CSVs: Comma
Separated Values. In a CSV, the data are delimited* by commas. Using `alt()` combinators and the delimited sequence
combinator, we can build a parser that matches the string that we'll need for or red, blue, green block parsers:

*mostly

```Rust
delimited(space0, alt((red, blue, green)), alt((tag(","), tag(""))));
```

Here's how we get a string to feed to the round parser:

```Rust
terminated(
        take_till(|c| c == ';' || c == '\n'),
        take_while(|c| c == ';' || c == ' '),
    );
```

We're using take_till and take_while. Inside each function is a closure the returns a boolean. The fancy term for
this is predicate. The difference between take_till and take_while is what they do in respond to their predicate:
Take till gives us back a slice up _until_ a predicate is met. So here, it will return a slice until a characer is
either a newline or a semicolon. On the other hand, take_while will keep matching a slice as long as its predicate
is met.

A round is a list of blocks:
```Rust
struct Round {
    blocks: Vec<Block>,
}
```
Nom has a parser for making such lists: [many1](https://docs.rs/nom/latest/nom/multi/fn.many1.html). It pushes results that match its parser into a standard
Rust vec. Putting this together we have:
```Rust
fn round(input: &str) -> IResult<&str, Round> {
    let mut round_parser = terminated(
        take_till(|c| c == ';' || c == '\n'),
        take_while(|c| c == ';' || c == ' '),
    );
    let block_parser = delimited(space0, alt((red, blue, green)), alt((tag(","), tag(""))));
    let (remaining, round_string) = round_parser(input)?;
    let (_, blocks) = many1(block_parser)(round_string)?;
    Ok((remaining, Round { blocks }))
}
```

Lastly, a game is a list of rounds.

```Rust
fn parse_game(input: &str) -> IResult<&str, Game> {
    let mut header_parser = delimited(tag("Game "), u32, tag(": "));
    let (game_string, id) = header_parser(input)?;
    let (remaining, rounds) = many1(round)(game_string)?;
    Ok((remaining, Game { id, rounds }))
}
```
When I print the resulting structs with the debug macro, I see that I'm getting the data that I expected, so my parser is done.


### Further Reading
I highly recommend the [Nominomicon](https://tfpk.github.io/nominomicon/). I find it takes the approach of building up its explanation of Nom slowly and with examples.

I also think [chris biscardi](https://www.youtube.com/watch?v=JOgQMjpGum0)'s 2023 AoC series was extremely detailed and useful.

[My AoC Submissions](https://github.com/TheCarton/AoC2023). You may need to read a few parsers before you get enough of an idea to write your own.

