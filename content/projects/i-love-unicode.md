+++
title = "I � Unicode"
date = 2025-10-05
weight = 1
template = "post.html"

[taxonomies]
tags = ["unicode", "emoji", "web", "presentation", "typography", "internationalization"]

[extra]
author = "Dominic Lüchinger"
repo_view = false
+++

_A curated collection of fascinating, funny, weird, and strange Unicode phenomena from my talk at the
[Webnesday St. Gallen Meetup](https://www.meetup.com/webnesday/events/310984075)._

**[📊 View the presentation slides](https://docs.google.com/presentation/d/1O9vH5F2YPh4oKDq2FriI_-bKFR4zRxDH_pLrsvIPRc8/edit?usp=sharing)**

Special thanks to [Pascal Helfenstein](https://www.linkedin.com/in/pascal-helfenstein/) for organizing this
fantastic event, [Frontify](https://www.frontify.com/) for hosting us, and my fellow speakers Erdem (keyboards) and
[Christoph Bühler](https://www.linkedin.com/in/christoph-b%C3%BChler-a3a262270/) (hypecycles) for making it such an
engaging evening!

Unicode is far more than just "international characters"—it's a deep rabbit hole of linguistic history, cultural
nuance, and technical complexity that continues to surprise developers daily. This page serves as an extended resource
for my presentation, featuring real-world Unicode gotchas, security implications, and delightfully bizarre examples.

## Getting Started with Unicode

Before diving into the weird and wonderful world of Unicode oddities, let's start with some excellent
introductions to Unicode basics:

### Essential Reading

These articles provide a solid foundation for understanding Unicode:

- **[The Absolute Minimum Every Software Developer Must Know About Unicode in 2023 (Still No
  Excuses!)](https://tonsky.me/blog/unicode/)** by Nikita Prokopov
  A modern, practical guide that covers what's changed since Joel Spolsky's famous 2003 article. Key insight: "In
  2023, it's no longer a question: with a 98% probability, it's UTF-8. Finally! We can stick our heads in the sand
  again!" Covers UTF-8, grapheme clusters, normalization, and why `"🤦🏼‍♂️".length` gives different answers in
  different programming languages.

- **[An Introduction to Unicode](https://www.aleksandrhovhannisyan.com/blog/introduction-to-unicode/)** by Aleksandr
  Hovhannisyan
  A comprehensive deep-dive into Unicode and UTF-8 encoding with mathematical explanations. Covers the history from
  ASCII to Unicode, how UTF-8 works under the hood with bitwise operations, character boundaries, and
  self-synchronization. Perfect for understanding the technical details of how UTF-8 encoding actually works.

- **[Unicode for Curious Developers](https://juliensobczak.com/inspect/2021/06/19/unicode-for-curious-developers/)**
  by Julien Sobczak
  An incredibly detailed guide covering the complete story of Unicode from ancient cave paintings to modern emojis.
  Explores the Unicode Standard, character database, encoding forms (UTF-8, UTF-16, UTF-32), and implementation
  details in various programming languages. Includes extensive code examples and practical guidance for developers.

- **[Unicode in Five Minutes](https://richardjharris.github.io/unicode-in-five-minutes/)** by Richard Harris
  A concise but comprehensive overview covering normalization, casefolding, sorting, encodings, and practical
  Unicode issues. Excellent coverage of "gotchas" like grapheme clusters, variation selectors, and internationalized
  domain names. Great for developers who need practical Unicode knowledge quickly.

These articles will help you understand the fundamentals before we explore the more unusual aspects of Unicode below.

## 🐍 Real-World Unicode Gotchas

### The Hyphen That Broke Everything

A perfect example comes from [Joseph Carboni's experience](https://pyatl.dev/2024/09/01/bitten-by-unicode/) parsing
financial data. He was extracting dollar amounts from PDFs, expecting negative values to have hyphens. His regex
`[^-.0-9]` should have preserved hyphens, but negative values kept coming out positive!

The culprit? Two visually identical but different characters:

- **HYPHEN-MINUS** (U+002D): `-` (the one on your keyboard)
- **HYPHEN** (U+2010): `‐` (the "proper" typographic hyphen)

```python
import unicodedata

# These look identical but aren't!
keyboard_hyphen = "-"
typographic_hyphen = "‐"

print(unicodedata.name(keyboard_hyphen))  # HYPHEN-MINUS
print(unicodedata.name(typographic_hyphen))  # HYPHEN

# The solution: Use Unicode categories
def is_hyphen(char: str) -> bool:
    return unicodedata.category(char) == 'Pd'  # Punctuation, dash
```

[Discussion on Hacker News](https://news.ycombinator.com/item?id=41485001) reveals this is surprisingly common!

### The Greek Question Mark Prank

Replace a semicolon (`;`) with a Greek question mark (`;`) in someone's code and watch them go insane trying to find
the syntax error. They're visually identical but completely different Unicode characters!

- Semicolon: U+003B (`;`)
- Greek Question Mark: U+037E (`;`)

Tools like [mimic](https://github.com/reinderien/mimic) can help identify these confusables.
[HN discussion](https://news.ycombinator.com/item?id=10437619)

## 🌍 Programming in Other Scripts

### قلب (Qalb) - Arabic Programming Language

[قلب](https://nas.sr/%D9%82%D9%84%D8%A8/) is a programming language that explores the role of human culture in coding
by using Arabic script. It demonstrates how deeply embedded English/Latin assumptions are in our programming tools and
thinking.

## 🎭 Security Implications

### IDN Homograph Attacks

[IDN Homograph attacks](https://en.m.wikipedia.org/wiki/IDN_homograph_attack) exploit visually similar characters from
different scripts to create deceptive domain names:

- `аpple.com` (using Cyrillic 'а' instead of Latin 'a')
- `goog1е.com` (using Cyrillic 'е' instead of Latin 'e')

These attacks rely on **Punycode encoding**, which converts Unicode domain names to ASCII. The browser might show the
Unicode version, but the actual domain uses encoded ASCII like `xn--pple-43d.com`.

### Right-to-Left Override Attacks

Using the Right-to-Left Override character (U+202E), attackers can make malicious files appear safe:

```text
document‮.txt.exe
```

This appears as `document.exe.txt` but is actually an executable file! The RLO character reverses the text display.

Example: `‮http://www.example.com?site/moc.elgoog.www//:ptth` - try copying this URL!

## 🔤 Extreme Unicode Examples

### The 15KB Character

There exists a Unicode character that takes up approximately 15,000 bytes when encoded. You can calculate character
byte sizes at [mothereff.in/byte-counter](https://mothereff.in/byte-counter).
[Reddit source](https://www.reddit.com/r/Unicode/comments/mpakgk/30000_characters_in_1_dot_you_can_fact_check_me/)

### ꙮ - The Multiocular O

Meet `ꙮ` (U+A66E) - the Cyrillic letter multiocular O with **many eyes**!

- [Unicode proposal document](https://www.unicode.org/wg2/docs/n5170-multiocular-o.pdf)
- [Neatorama article](https://www.neatorama.com/2022/09/19/An-Important-Update-to-the-Multiocular-O/)
- [Wikipedia: Cyrillic O variants](https://en.wikipedia.org/wiki/Cyrillic_O_variants#Multiocular_O)

### 䨺 - The Character with the Most Strokes

`䨺` is the [Taito kanji](<https://en.m.wikipedia.org/wiki/Taito_(kanji)>) with a whopping 84 strokes, making it one of
the most complex characters in Unicode.

### Ancient Scripts

- **힘** (Korean): [U+D798](https://www.compart.com/en/unicode/U+D798) -
  [Graphemica](https://graphemica.com/%ED%9E%98)
- **𓈝** (Egyptian Hieroglyph): Ancient Egyptian cow symbol
- **𒐫** (Cuneiform): [Cuneiform number](https://unicodes.jessetane.com/%F0%92%90%AB) from the
  [Cuneiform Numbers and Punctuation](https://en.wikipedia.org/wiki/Cuneiform_Numbers_and_Punctuation) block

### Egyptian Hieroglyphs with... Personality

The hieroglyphs 𓂸𓂹𓂺 have caused quite a stir in the Unicode community. Adrian Kennard wrote about them in
[Unicode Dicks](https://www.revk.uk/2018/10/unicode-dicks.html) with an entertaining
[Hacker News discussion](https://news.ycombinator.com/item?id=18346922).

## 📚 Essential Reading for Developers

### The Modern Unicode Developer's Bible

[**The Absolute Minimum Every Software Developer Must Know About Unicode in 2023**](https://tonsky.me/blog/unicode/) by
Nikita Prokopov is hands-down the best comprehensive Unicode guide for developers. This brilliant article covers:

#### Key Insights from Tonsky's Guide

- **UTF-8 has won**: 98% of the web uses UTF-8, so we can finally stop worrying about encoding detection
- **Grapheme clusters matter more than code points**: What users see as "one character" often spans multiple code points
- **String length is broken in most languages**: Only Swift and Elixir get `"🤦🏼‍♂️".length` right (answer: 1)
- **Use Unicode libraries for everything**: Even basic operations like `strlen`, `indexOf`, and `substring` need proper
  Unicode handling
- **Unicode updates yearly**: Rules change, new emoji are added, and your app needs to keep up

#### Real-World Developer Problems It Solves

- Why `"🤦🏼‍♂️"` reports different lengths in different languages (Python: 5, JavaScript: 7, Rust: 17 bytes)
- How normalization prevents `"Å" !== "Å" !== "Å"` comparison failures
- When `String.toLowerCase()` requires a Locale parameter (Turkish `i`/`I` problem)
- Why UTF-16 surrogate pairs still matter for JavaScript/Java/.NET developers

#### Essential Takeaways

1. **Extended Grapheme Clusters** are what humans think of as "characters"
2. **Normalization** is required before any string comparison
3. **Locale matters** for case conversion and rendering
4. **Even English text** uses Unicode beyond ASCII (curly quotes, em dashes, café)

This guide perfectly complements the weird examples on this page by explaining the underlying technical foundations
that make Unicode both powerful and occasionally maddening.

### More Essential Reading

**[Dark Corners of Unicode](https://eev.ee/blog/2015/09/12/dark-corners-of-unicode/)** by eevee - A brilliant deep dive
into the practical problems you'll encounter with Unicode in the real world:

- **Terminal rendering nightmares**: Why emoji overlap text in VTE and cursor positioning breaks in Konsole
- **JavaScript's broken string type**: How `"💣".length` returns 2 because JavaScript uses UTF-16 surrogate pairs
- **The wcwidth() disaster**: Different implementations report different character widths, breaking text everywhere
- **Sorting is impossible**: German "ß" vs "ss", Turkish dotless "ı", and why normalization isn't a silver bullet
- **There's no such thing as emoji**: The arbitrary definition of what counts as emoji and why fonts matter

**[I Can Text You A Pile of Poo, But I Can't Write My Name](https://modelviewculture.com/pieces/i-can-text-you-a-pile-of-poo-but-i-cant-write-my-name)**
by Aditya Mukerjee - A powerful critique of Unicode's cultural and representational problems:

- **Second-class languages**: Bengali (7th most spoken language) missing basic characters for decades
- **Han Unification controversy**: Forcing Chinese, Japanese, and Korean into shared character sets
- **Colonial echoes**: How Unicode Consortium's composition reflects historical power imbalances
- **Emoji prioritization**: 1,000 emoji characters while people can't write their own names correctly

## 💻 Developer Deep Dives

### Monospace Fonts and Unicode Ligatures

**[Shaping Ligatures in Monospace Fonts](https://joshleeb.com/posts/monospace-ligatures.html)** by Josh Leeb explores
the technical challenges of implementing Unicode ligatures in code editors:

- **The ligature spacer problem**: How `"#{"` becomes 4 glyphs instead of 3 when shaped
- **Invisible glyphs**: The mysterious "LIGSPACE" character with 2×0 pixel dimensions
- **Monospace constraints**: Why fonts use phantom spacer glyphs to maintain fixed-width requirements
- **Rendering complexity**: Real-world text shaping is never as simple as "one character = one glyph"

This perfectly illustrates how even "simple" programming contexts involve deep Unicode complexity.

## 👹 Zalgo Text - When Unicode Goes Wrong

[Zalgo text](https://en.m.wikipedia.org/wiki/Zalgo_text) uses combining diacritics to create chaotic, "corrupted"
looking text:

```text
H̵̛͕̞̦̰̜͍̰̥̟͆̏͂̌͑ͅä̷͔̟͓̬̯̟͍̭͉͈̮͙̣̯̬͚̞̭̍̀̾͠m̴̡̧̛̝̯̹̗̹̤̲̺̟̥̈̏͊̔̑̍͆̌̀̚͝͝b̴̢̢̫̝̠̗̼̬̻̮̺̭͔̘͑̆̎̚ư̵̧̡̥̙̭̿̈̀̒̐̊͒͑r̷̡̡̲̼̖͎̫̮̜͇̬͌͘g̷̹͍͎̬͕͓͕̐̃̈́̓̆̚͝ẻ̵̡̼̬̥̹͇̭͔̯̉͛̈́̕r̸̮̖̻̮̣̗͚͖̝̂͌̾̓̀̿̔̀͋̈́͌̈́̋͜
```

## 🔧 Developer Tools and Libraries

### String Length: Not What You Think

Counting emoji and Unicode characters is trickier than it seems:
[How to Count Emojis with JavaScript](https://cestoliv.com/blog/how-to-count-emojis-with-javascript/)

### Libraries and Tools

- **PHP**: [Symfony String Component](https://symfony.com/doc/current/string.html) - Unicode-aware string handling
- **Python**: Use `u'...'` for Unicode-aware strings (Python 2) or just strings in Python 3+
- **Fonts**: [Symbola](https://localfonts.eu/freefonts/greek-free-fonts/unicode-fonts-for-ancient-scripts/symbola/) -
  Comprehensive Unicode font for ancient scripts

### Fun Unicode Tools

#### Regional Indicator Generator

[LingoJam Regional Indicator Generator](https://lingojam.com/RegionalIndicator) - Convert text to spaced-out flag
emoji letters like 🇨 🇴 🇩 🇪.

**How it works:** Regional Indicator symbols (U+1F1E6-U+1F1FF) represent letters A-Z and are intended for ISO 3166-1
country codes. When two valid codes are adjacent, they become flag emoji (🇺🇸, 🇨🇦), but when spaced out, they
create stylized text effects popular on Discord and social media.

**The Unicode trick:** Each "letter" is actually U+1F1E8 REGIONAL INDICATOR SYMBOL LETTER C, etc. - originally
designed for encoding country flags, but cleverly repurposed for decorative text.

### Unicode Mirror Characters

Some characters have "mirrored" versions for right-to-left text:
[Stack Overflow discussion](https://stackoverflow.com/questions/3115204/unicode-mirror-character)

## 📊 Unicode Visualization

### The Big Picture

[Ian Albert's Unicode Chart](https://ian-albert.com/unicode_chart/#download) - A massive visual representation of the
entire Unicode space. Perfect for understanding the scale and organization of Unicode blocks.

## 🐦 Twitter/X's Unicode Reality Check

[Twitter's character counting documentation](https://docs.x.com/fundamentals/counting-characters) reveals the messy
reality of Unicode in production systems at massive scale:

### The "280 Character" Lie

Twitter's character limit isn't actually 280 Unicode characters:

- **Most characters count as 1**: Latin-1, basic punctuation, directional marks (U+0000-U+10FF)
- **CJK characters count as 2**: Chinese/Japanese/Korean users get only 140 characters max
- **ALL emoji count as 2**: Even simple ones like 👾, regardless of underlying complexity
- **Complex emoji still count as 2**: 👨‍👩‍👧‍👦 (7 Unicode code points) = 2 Twitter "characters"

### Unicode Normalization in Production

Twitter normalizes all text to **NFC (Normalization Form C)** before counting:

```text
"café" (composed): 0x63 0x61 0x66 0xC3 0xA9 = 4 characters
"café" (decomposed): 0x63 0x61 0x66 0x65 0xCC 0x81 = still 4 characters (after NFC)
```

### The t.co URL Hack

All URLs become exactly **23 characters** regardless of actual length:

- `https://example.com` = 23 characters
- `https://reallyreallylongdomainname.com/with/many/paths` = also 23 characters

### Zero Width Joiner Magic

Twitter recognizes emoji sequences using Zero Width Joiner (U+200D) but counts them as 2 characters total:

- 👨‍🎤 = 👨 + ZWJ + 🎤 = 2 Twitter characters (not 3 Unicode code points)

This demonstrates how even major platforms struggle with Unicode complexity and create arbitrary rules to make
things work. It also shows why Unicode awareness matters—CJK users get half the character limit of English users!

## 🎯 Key Takeaways

1. **Never assume character equality** - Always normalize and compare properly
2. **Security matters** - Unicode can be weaponized for phishing and attacks
3. **Cultural context is everything** - Scripts carry deep cultural meaning
4. **Test with real data** - PDF extractions and copy-paste introduce surprising characters
5. **Use proper libraries** - Don't roll your own Unicode handling

Unicode isn't just a technical specification—it's a reflection of human linguistic diversity, complete with all the
complexity, beauty, and occasional chaos that entails. Every weird edge case has a story, often rooted in centuries
of cultural and typographic history.

## 🎥 Recommended Unicode Talks

**[Unicode: What Everyone Should Know](https://www.youtube.com/watch?v=gd5uJ7Nlvvo)** - A wonderful deep-dive talk
exploring Unicode fundamentals, practical challenges, and real-world implications for developers. Perfect complement
to the resources and examples on this page.

**[Unicode Talks Collection](https://www.youtube.com/watch?v=vpSkBV5vydg)** - Additional insights into Unicode
complexity and practical developer challenges.

**[Unicode Technical Deep-Dive](https://www.youtube.com/watch?v=ut74oHojxqo)** - Further exploration of Unicode
implementation details and real-world scenarios.

**[Unicode Advanced Topics](https://www.youtube.com/watch?v=uTJoJtNYcaQ)** - Extended discussion of Unicode
complexities and advanced implementation considerations.

---

_Found a great Unicode oddity or have a war story to share? Unicode never stops surprising us!_
