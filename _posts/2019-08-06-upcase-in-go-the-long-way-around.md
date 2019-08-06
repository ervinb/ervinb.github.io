---
layout: article
title: Checking a character's case in Go, the long way around (0's and 1's included)
tags: go golang
key: go-upcase
---

I'm slowly picking up Go, and ran across a [Hacker Rank](https://www.hackerrank.com/) task: count the number of uppercase letters, in an
input, which is written in CamelCase. For example, thisStringWouldYield 3.

It's easy as it sounds: you go through each character and increase a counter if it's in uppercase. ([playground](https://play.golang.org/p/s6XtaXjwGg2))

```golang
func camelCount(in string) int {
	counter := 0

	for _, ch := range in {
	  if (unicode.IsUpper(ch)){
	    counter++
	  }
	}
	
  return counter
}
```

... but as regexes, exiting Vim, and [character encoding](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/) are often a weak point of
many of us, I've decided to dig deeper, and see how Go implements `IsUpper`. It turned out
to be a good refresher about some basics, which often get overlooked or are taken for granted.

## Understanding the problem

Let me throw in a couple of words, and see how many of them we know, and how many of them we think we know: ASCII, ANSI, Unicode, encoding, UTF8, UTF16, code point, rune. I will give a brief description below,
but reading [Joel's article](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/) is definitely a better way to learn about them:

- ASCII: a set of 128 characters, consisting of the English alphabet, control characters, numbers, punctation marks. A character maps directly to a value in memory: `F -> 01000110 (70)`. Uses 1 byte (8 bits), so this leaves 128 bits up for grabs. This is where ANSI comes in.
- ANSI: a set of 256 characters, where the first 128 are from ASCII, and the second part was used for special characters like `ő` and `Þ`. As you might imagine, there are more than 128 characters other than English ones,
so the upper 128 slots were divided into code pages (Arabic, Greek, etc.).
- Unicode: a successful attempt to unify all possible characters in a single character set. The code `U+0048` points to letter `H` - we call this a `code point`. This is then encoded to memory: `H -> U+0048 -utf8-> 110000 (48)`.
- encoding: storing a code point in memory. There are hundreds of types, but most of them can represent only a portion of the Unicode character set. This is where UTF-8 comes in.
- UTF-8: an encoding format for the *whole* Unicode character set, which stores the first 127 characters in 1 byte (8 bits). This makes it compatible with ASCII and ANSI up to the first 127 characters. Characters above this
are stored in 2 to 6 bytes.

Back to the task at hand of checking if a character is upper case. "A" is for example encoded to the decimal 41, where lowercase "a" is encoded to 61. So, a question arises, is there a function (like adding/subtracting 20)
which can easily change the character's case? The answer is more simple, and more complicated at the same time.

We don't need any fancy formula, because everything is hardcoded into programming language. There's a huge hardcoded table of characters, from which we get what we need, and continue with our day.

The "get what we need" is where it gets interesting. Let's dive into the internals of Go.

I will break down the code examples as much as I can, to assure a firm base for understanding the implementation. This will
include some Go and general programming concepts. Let's see how `isUpper()` works under the hood.

## Looking under isUpper()

The implementation is suspiciously short: 


```golang
func IsUpper(r rune) bool {
	if uint32(r) <= MaxLatin1 {
		return properties[uint8(r)] & pLmask == pLu
	}

	return isExcludingLatin(Upper, r)
}
```

We will focus on understanding line 3:

```golang
return properties[uint8(r)] & pLmask == pLu
```

We're accessing an array called `properties` with an 8-bit integer index, derived from the rune (code point), have some kind of a mask, and `pLu` whatever that is. Making sense of both sides of the equality check is up next.


## Flags and bitmasks

If we look up the definition of `pLmask` ((src/unicode/graphic.go)[https://golang.org/src/unicode/graphic.go#8]), your first reaction might be to switch to the YouTube tab you have sitting next to
the current one, but let's fight that urge and see what we have here. 

```golang
const (
  _ = 1 << iota
  ...
  pLl
  pLu
  ...
  pLmask = pLu | pLl
```

Yes, this is how the Pandora's box looks like. If you're fluent in Go and bitwise operations, switch to that YouTube tab until the rest of us finish here.

*Q*: Why aren't all the elements defined, if these are constants?

*A*: They are. The missing type and value just means to repeat the line above as long as it doesn't encounter an equals sign. [link to iota]
A property of Go's `iota` is to auto-increment as it moves through the list with each new line. 

```golang
const (
  a = 0 + iota   # => 0 + 0 = 0
  b              # => 0 + 1 = 1
  c              # => 0 + 2 = 2
)
```


*Q*: _tries to skim over '_ = 1 << iota'_

*A*: The elephant at the top, known as a left bitwise shift, is one of the rarer operations you'll see in your everyday life, and it's like seeing a full kitchen sink - you reflexively look away.
Bitwise operations are usually used to represent flags. Pseudo code, describing ships and a vacation coming up; we've arrived to 0's and 1's, but it's fun I promise. 

Let's take the following flags, to help us define our `vacationCriteria`:

```golang
slipperyDeck   = 1 # binary: 0001
onboardPool    = 2 # binary: 0010
captainOnBoard = 4 # binary: 0100
freeSnacks     = 8 # binary: 1000

```

The values are not arbitrary: `1` in the binary representation of the flag is always on a different place, and it never clashes with another flag.
If we were looking for a ship with a swimming pool, and we're into slippery decks, we combine `onboardPool` and `slipperyDeck`:

```golang
vacationCriteria = slipperyDeck | onboardPool # which is equal to: 0001 | 0010 = 0011
```

Flags are combined with the logical `OR` operation, represented with `|`. Also our `vacationCriteria` is what is called a bitmask. We use bitmasks, to see if a flag is set or not, by using the logical `AND` (`&`) operation.
Often, this is called `extracting the flag`.

Now, we found've a potential ship. Let's check if it fits our needs:

```golang
ship = 15 # binary: 1111, pretty flashed out ship
packBags() if (ship & vacationCriteria) # 1111 & 0011 = 0011, which is 'true' so let's packBags()
```

Conveniently, it also has a `captainOnBoard` and `freeSnacks`.

When defining the flags, to conserve our brain cycles from moving the binary `1` without clashing with other flags, we use a bitwise left shift `<<`. It does exactly we've done above when we defined the set of flags.
Combined with `iota` (which auto-increments with each line), we always get a new variable, which exactly avoids clashing with the previous one. In other words, with each new line, the `1` in the binary representation of the value, is always one position further from the end than in the previous line.
(An interesting property of this operation is that the end result of `x << y` can be calculated with `x * 2^y`.)

Back to the flag definitions in `unicode/graphic.go`.

```golang
const (
  _ = 1 << iota       // 1 << 0 = 00001 << 0 => 00000001 (1 * 2^0 = 1)
  ...                 // four other flags defined here
  pLl                 // 1 << 5 = 00001 << 5 => 00100000 (1 * 2^5 = 32)
  pLu                 // 1 << 6 = 00001 << 6 => 01000000 (1 * 2^6 = 64)
  ...
  pLmask = pLu | pLl  //                     => 01100000 (32 + 64 = 96)
```

At this point, we're masters of this snippet.
To make sense of the variable names here: the capital letter refers to Unicode categories: Letter, Character, Space, Other. The ending lowercase letter is, as you probably already guessed, referring to `lowercase` or `uppercase`.

So `pLmask` is a bitmask, which has the uppercase and lowercase flags set to true. By applying a bitwise `&` and a character's `properties`,
we can check if the character has either the lowercase or uppercase flag set.


## Making sense of it all

We're now armed to decypher the line from which we started, in 3 quick strokes.

```golang
properties[r] & pLmask == pLu

```

1. `properties` is an array which stores flags, like our `vacationCriteria` from before. For example at position 41, you'll find:

    ```golang
    properties = [MaxLatin1 + 1]uint8 {
      ...
      0x41: pLu | pp, // 'A'
      ...
    }
    ```

    Conveniently, the position corresponds to the character's number representation in the Unicode table. With `properties[uint(A)]` we get the value of the `pLu | pp` flags.
    `pp` means "printable" and it doesn't interfere with our check for `pLu`, as it's not present in `pLmask`.


    ```golang
    properties[uint("A")] = pLu | pp // 01000000 | 10000000 => 11000000 
    ```

    So based on the above, we know that the capital "A" is printable and that it's uppercase. In the sections below, instead of `properties[]`, we will write out `(pLu | pp)`, so that the logical operations are more clear.

2. The next thing in line is `& pLmask`. With it we check if the value of the given `properties` element, has the `pLmask` flags activated, like we checked if a ship is suitable for our vacation.

    ```golang
    (pLu | pp) & pLmask // 11000000 & 01100000 => 010000000
    ```

    Again, `(pLu | pp)` is derived from step 1, and it's the value of `properties[uint("A")]`.

3. Lastly, we check if the extracted value is exactly `pLu` ie. is the character is exactly upper case. It can't be `pLu` and `pLl` at the same time. That would make a strange letter.

    ```golang
    ((pLu | pp) & pLmask) == pLu // 010000000 == 010000000, true
    ```

    Now we can answer the question: is the letter A uppercase?

    ```
    If a letter has the 'pLu' flag set, it is an uppercase letter.
    ```

---