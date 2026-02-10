---
title: "So, I Thought I Knew Strings in Go: A Jon Snow Moment"
date: 2025-06-22
description: "That’s me with Go strings. I thought I had them figured out after coding with them for a while. Turns out, I was in for a surprise. Let’s dive into my"
author: "Bhawesh Kumar Singh"
image: "images/blog/strings-in-go.jpg"
categories: ["Go", "Programming"]
medium_url: "https://medium.com/@bhaweshkumarsingh/so-i-thought-i-knew-strings-in-go-a-jon-snow-moment-cb67c8d2d1b6"
---

*Originally published on [Medium](https://medium.com/@bhaweshkumarsingh/so-i-thought-i-knew-strings-in-go-a-jon-snow-moment-cb67c8d2d1b6)*

That’s me with Go strings. I thought I had them figured out after coding with them for a while. Turns out, I was in for a surprise. Let’s dive into my journey of unraveling the mysteries of strings in Go, complete with code snippets, some head-scratching moments, and lessons learned. Buckle up!
![](https://cdn-images-1.medium.com/max/500/0*a_nnx953OXMn3Buv.jpg)
### The Basics: What Is a String in Go?

When you initialize a string like this:

```go
value := "this is a test string"
```

You might think it’s just a sequence of characters. Close, but not quite. In Go, a string is internally a **read-only slice of bytes** with a length. Simplified, it looks like this:

```go
type string struct {
	data *([]byte)
	len  int
}
```

I had a vague idea about this, but the details? Not so much. Let’s explore how to work with strings, why things get weird with non-ASCII characters, and how to avoid falling on your face.

### Accessing String Contents: The ASCII Trap

My first instinct was to access a string’s characters by index, like value[0]. For ASCII strings, this works fine because each character is one byte. But throw in a Unicode character like é, and things get spicy.

Here’s an example to show what I mean:

```go
package main

import "fmt"

func main() {
	value := "thé" // Contains ASCII ('t', 'h') and non-ASCII ('é')
	fmt.Println(value[1])        // Output: 104
	fmt.Printf("%c\n", value[1]) // Output: h
	fmt.Println(value[2])        // Output: 195
	fmt.Printf("%c\n", value[2]) // Output: Ã (not é!)
	fmt.Println(string(value[2])) // Output: Ã (still not é!)
}
```

### What’s Going On Here?

This isn’t tomfoolery — it’s how Go handles strings. Strings in Go are encoded in **UTF-8**, a variable-length encoding where:
- ASCII characters (like t or h) are **1 byte**.
- Non-ASCII characters (like é) can be **1 to 4 bytes** (in this case, 2 bytes).

Let’s break down the string “thé”:
![](https://cdn-images-1.medium.com/max/1010/1*GcWQuqqdZipg8ZyhrZXfew.png)

The byte array for “thé” is:

```text
[0x74, 0x68, 0xC3, 0xA9]
  t      h         é
```

When you do value[1], you get the byte at index 1 (0x68, or 104), which corresponds to h. But value[2] gives you 0xC3 (195), the first byte of é. Printing it as a character (%c) or **converting it to a string (string(value[2])) gives Ã because it’s only half of é’s UTF-8 encoding**.
![](https://cdn-images-1.medium.com/max/798/1*mBmU25ZSiEx6kLdzDjw8fA.png)
Mind blown !

The default format for fmt.Println(value[1]) is %d (integer), which is why you see 104. Using %c gives the character h.

Lesson? Indexing gives you **bytes**, not characters.

### Updating Strings: The Immutable Wall

Now that I understood the byte array, I thought I could update a string like this:

```go
package main

func main() {
	value := "this is a test string"
	value[0] = 'h' // Error: cannot assign to value[0]
}
```

Nope! Strings in Go are **immutable**. You can’t modify the internal byte array. The data pointer in the string struct is read-only, and any change creates a **new string**. For example:

```go
k := value[1:] // "his is a test string"
```

Here, k is a new string, not a modified version of value.

To update a string, convert it to a []byte, modify the slice, and convert back:

```go
package main

import "fmt"

func main() {
	value := "this is a test string"
	byteValue := []byte(value)
	byteValue[0] = 'T' // This works!
	newStringValue := string(byteValue)
	fmt.Println(newStringValue) // Output: This is a test string
}
```

### Iterating Over Strings: Bytes vs. Runes

There are two ways to iterate over a string in Go:
- **Byte indexing**: value[i] gives the byte at index i.
- **Range loop**: for i, r := range value gives the **runes** (Unicode code points).

Here’s the difference in action:

```go
package main

import "fmt"

func main() {
	value := "thé" // ASCII ('t', 'h') and non-ASCII ('é')

	// Byte iteration
	fmt.Println("Bytes:")
	for i := 0; i < len(value); i++ {
		fmt.Printf("Byte at index %d: %d (%c)\n", i, value[i], value[i])
	}

	// Rune iteration
	fmt.Println("\nRunes:")
	for i, r := range value {
		fmt.Printf("Rune at index %d: %d (%c)\n", i, r, r)
	}
}
```

**Output**:

```text
Bytes:
Byte at index 0: 116 (t)
Byte at index 1: 104 (h)
Byte at index 2: 195 (Ã)
Byte at index 3: 169 (©)
```

```text
Runes:
Rune at index 0: 116 (t)
Rune at index 1: 104 (h)
Rune at index 2: 233 (é)
```

Byte iteration gives you raw bytes, **which can split multi-byte characters** like é (hence the Ã and ©).

The **range loop, however, iterates over runes**, correctly handling é as a single character (U+00E9, or 233).

### String Length: Bytes vs. Characters

The len() function returns the number of **bytes**, not characters:

```go
value := "thé"
fmt.Println(len(value)) // 4 (bytes: t, h, C3, A9)
```

```go
asciiStringValue := "the"
fmt.Println(len(asciiStringValue)) // 3 (bytes: t, h, e)
```

To count **characters** (runes), use utf8.RuneCountInString:

```go
import "unicode/utf8"

value := "thé"
fmt.Println(utf8.RuneCountInString(value)) // 3 (runes: t, h, é)
```

### Conversions: Bytes, Runes, and Back

Converting a string to []byte copies its bytes:

```go
value := "thé"
bytes := []byte(value) // [116, 104, 195, 169]
```

Converting to []rune decodes the bytes into Unicode code points:

```go
runes := []rune(value) // [116, 104, 233]
fmt.Println(string(runes[2])) // "é"
```

### Unicode and UTF-8: The Deep Dive

**Unicode** assigns a unique integer (code point) to every character. For example, é is U+00E9 (233 in decimal). **UTF-8** is a way to encode these code points as 1–4 bytes. ASCII characters use 1 byte, while others, like é, use more.

This explains why value[2] gave us 195 (part of é’s UTF-8 encoding) and why range loops are better—they decode UTF-8 into runes.

### Practical Tips to Avoid Jon Snow Moments

Here’s what I’ve learned to handle strings like a pro:
- **Use for _, r := range s for iteration**: It’s the idiomatic way to process characters (runes).
- **Avoid len(s) for character counting**: Use utf8.RuneCountInString for runes.
- **Convert to []rune for indexing**: Need to access characters? Use a rune slice.
- **Be explicit about bytes vs. runes**: Use []byte for I/O and []rune for character manipulation.
- **Leverage unicode/utf8**: For complex UTF-8 tasks, use functions like utf8.DecodeRune.

Strings in Go are deceptively simple but packed with nuance. They’re immutable, UTF-8 encoded, and require careful handling of bytes vs. runes. Now that I’ve survived my Jon Snow moment, I’m better equipped to handle strings. Hopefully, this post saves you from the same surprises!

Got any string-related war stories? Share them below!
