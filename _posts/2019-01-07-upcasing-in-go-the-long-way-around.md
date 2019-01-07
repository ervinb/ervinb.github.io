# Checking a characters case in Go, the long way around (0's and 1's included)

Changing the casing of a word or a letter, has to be one of the most common thing we do,
as developers. The action itself seems to be trivial, and we hardly pause to think about
how it's done. Let's do just that. Let's throw in 1's and 0's as well, for good measure.

a -> no
A -> yes

It's that easy. But that arrow hides a few interesting things, from which we can learn a thing or two.

## Motivation

I'm slowly picking up Go, and ran across a Hacker Rank task: count the number of uppercase letters, in an
input which is written in CamelCase. For example, thisStringWouldYield 3.

The solution is trivial ([playground](https://play.golang.org/p/s6XtaXjwGg2)),

```Go
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

... but as regexes, character encodings [link to SO dev post], and exitting Vim are often a weak point of
many of us, I've decided to dig deeper, and see how Go implements the `isUpcase` method (function?).


### Understanding the problem

Encoding tldr;

All these characters on your screen are defined in the Unicode table, and are encoded in UTF8. Each character has it's decimal representation, along with Hex or
any other format you fancy.

The upper case "A" is for example represented with a decimal 41 im the table; where lowercase "A" is at 61. So is there a function (like adding/subtracting 20)
which can easily change the character's case? The answer is more simple, and more complicated at the same time.

We don't need any fancy algorithm because everything is hardcoded. We have a huge table of hardcoded mappings, from which we get what we need, and continue with our day.

The "get what we need" is where it gets interesting.


I will break down the code examples as much as I can, to assure a firm base for understanding the implementation. This will
include some Golang and general programming concepts.

## isUpcase

The implementation is suspiciously 

```Go
func IsUpper(r rune) bool {
	if uint32(r) <= MaxLatin1 {
		return properties[uint8(r)] & pLmask == pLu
	}

	return isExcludingLatin(Upper, r)
}
```

We will focus on understanding line 2, in depth.

```Go
return properties[uint8(r)] & pLmask == pLu
```

We're accessing an array called `properties` with an 8-bit integer index derived from the character, have some kind of a bitmask, and `pLu` whatever that is.

Quick note on runes: A string is a sequence of bytes, which represent Unicode [code points](https://en.wikipedia.org/wiki/Code_point) (the 41 in `41 -> 'A'`) which are known as runes in Golang.


#### Flags and bitmasks

If we look up the definition of `pLmask` ((src/unicode/graphic.go)[https://golang.org/src/unicode/graphic.go#8]), the first reaction might be to switch to the YouTube tab you have sitting next to
the current one, but let's fight that urge and see what we have here. 

```
const (
  _ = 1 << iota
  ...
  pLl
  pLu
  ...
  pLmask = pLu | pLl
```

Yes, this is how Pandora's box looks like. If you're fluent in Go and bitwise operations, switch to that YouTube tab until the rest of us finish here.

*Q*: Why aren't all the elements defined, if these are constants?

*A*: They are. The missing type and value just means to repeat the line above as long as it doesn't encounter an equals sign. [link to iota]
A property `iota` is to auto-increment as it moves through the list with each new line. 

```Go
const (
  a = 0 + iota   # => 0 + 0 = 0
  b              # => 0 + 1 = 1
  c              # => 0 + 2 = 2
)
```


*Q*: _tries to skim over '_ = 1 << iota'_

*A*: The elefant at the top, known as left bitwise shift, is one of the rarer operations you see in your everyday life, and it's like seeing a full kitchen sink - you reflexively look away.
We've arrived to 0's and 1's, but it's fun I promise. 
Bitwise operations are usually used to represent flags. Pseudo code, describing ships and a vacation comming up.

```
slipperyDeck   = 1 # binary: 0001
onboardPool    = 2 # binary: 0010
captainOnBoard = 4 # binary: 0100
freeSnacks     = 8 # binary: 1000

```

The values are not arbitary: `1` in the binary representation of the flag is always on a different place, and it never clashes with another flag.
If we were looking for a ship with a swimming pool, and we're into slippery decks, we combine `onboardPool` with `slipperyDeck`:

```
vacationCriteria = slipperyDeck | onboardPool # 0001 | 0010 = 0011
```

Flags are combined with the logical `OR` operation, represented with `|`. Also our `vacationCriteria` is a bitmask. When used with logical `AND` (`&`), it shadows all other flags
and returns only the ones from the bitmask.

We found a potential ship. Let's check if it fits our needs.

```
ship = 15 // 1111
packBags if (ship & vacationCriteria) // 1111 & 0011 = 0011, which is 'true' => packBags
```

It does. Conveniently, it also has a `captainOnBoard` and `freeSnacks`.

When defining the flags, to spare our brain cycles from moving the binary `1` without clashing with other flags, we use bitwise left shift `<<`. It does exactly we've done above.
An interesting property of this operation is that the end result of `x << y` can be calculated with `x * 2^y`. Back to the flag definitions in `unicode/graphic.go`.

```
const (
  _ = 1 << iota       // 1 << 0 = 00001 << 0 => 00000001 (1 * 2^0 = 1)
  ...
  pLl                 // 1 << 5 = 00001 << 5 => 00100000 (1 * 2^5 = 32)
  pLu                 // 1 << 6 = 00001 << 5 => 01000000 (1 * 2^6 = 64)
  ...
  pLmask = pLu | pLl  //                     => 01100000 (32 + 64 = 96)
```

And to make sense of the variable namings: the capital letter refers to Unicode categories: Letter, Character, Space, Other.
The ending lowercase letter is, as you probably already guessed, is referring to `lowercase` or `uppercase`.


### Decyphering our line

We're now armed to decrypt the line from which we started, in 3 quick strokes.

```
properties[r] & pLmask == pLu

```

1. `properties` is an array which stores flags, like our `vacationCriteria` from before. For example at position 41, you'll find:

```
properties = [MaxLatin1 + 1]uint8 {
  ...
	0x41: pLu | pp, // 'A'
  ...
}
```

Conveniently, the position corresponds to the character's number representation in the Unicode table. With `properties[uint(A)]` we get the value of the `pLu | pp` flags.
`pp` means "printable" and it doesn't interfere with our check for `pLu`, as it's not present in `pLmask`.

```
properties[uint("A")] = pLu | pp // 01000000 | 10000000 => 11000000 
```

2. Applying `& pLmask` we 'extract' the `pLu`, like we checked if the ship is suitable for our vacation.

```
(pLu | pp) & pLmask // 11000000 & 01100000 => 010000000
```

3. Lastly, we check if the extracted value is exactly `pLu` ie. is the character is exactly upper case. It can't be `pLu` and `pLl` at the same time. That would make a strange letter.

```
((pLu | pp) & pLmask) == pLu // 010000000 == 010000000, true
```

go playground

What is an uppercase letter?

```
A letter which can be associated with the 'pLu' flag is an uppercase letter.
```

---

### End

"A"

Yay an uppercase A!