# N___ RegEx Library

## What is NRE?

A new regular expression library for Nim using PCRE to do the hard work.

## Why?

The [re.nim][] module that [Nim][] provides in it's standard library is
inadequate:

 - It provides only a limited number of captures, while the underling library
 (PCRE) allows an unlimited number.
 - Instead of having one proc that returns both the bounds and substring, it
 has one for the bounds and another for the substring.
 - If the splitting regex is empty (`""`), then it returns the input string
 instead of following [Perl][], [Javascript][], and [Java][]'s precedent of
 returning a list of each character (`"123".split(re"") == @["1", "2", "3"]`).

[re.nim]: http://nim-lang.org/re.html
[Nim]: http://nim-lang.org/
[Perl]: https://ideone.com/dDMjmz
[Javascript]: http://jsfiddle.net/xtcbxurg/
[Java]: https://ideone.com/hYJuJ5


# Documentation

## Construction

Creating a pattern is easy: `re"([0-9]+)"`. By default, the extended flag is
passed in order to encourage readable expressions, so `[0-9]+` is equivalent to
`[0-9] +  # foo`. If you'd like to pass your own flags, then `re(r"([0-9]+)",
"<flags>")` will work. Here is a list of the available flags:

  - `8` - treat both the pattern and subject as UTF8
  - `9` - prevents the pattern from being interpreted as UTF, no matter what
  - `A` - as if the pattern had a `^` at the beginning
  - `E` - DOLLAR_ENDONLY
  - `f` - fails if there is not a match on the first line
  - `i` - case insensitive
  - `m` - multi-line, `^` and `$` match the beginning and end of lines, not of
  the subject string
  - `N` - turn off auto-capture, `(?foo)` is necessary to capture.
  - `s` - `.` matches newline
  - `S` - study the pattern to hopefully improve performance. JIT is unspported
  at the moment.
  - `U` - expressions are not greedy by default. `?` can be added to a
  qualifier to make it greedy.
  - `u` - same as `8`
  - `W` - Unicode character properties
  - `X` - "Extra", character escapes without special meaning (`\w` vs. `\a`)
  are errors
  - `x` - extended, comments (`#`) and newlines are ignored (extended)
  - `Y` - pcre.NO_START_OPTIMIZE,
  - `<cr>` - newlines are separated by `\r`
  - `<crlf>` - newlines are separated by `\r\n` (Windows default)
  - `<lf>` - newlines are separated by `\n` (UNIX default)
  - `<anycrlf>` - newlines are separated by any of the above
  - `<any>` - newlines are separated by any of the above and Unicode newlines:
  > single characters VT (vertical tab, U+000B), FF (form feed, U+000C), NEL
  > (next line, U+0085), LS (line separator, U+2028), and PS (paragraph
  > separator, U+2029). For the 8-bit library, the last two are recognized
  > only in UTF-8 mode.
  - `<bsr_anycrlf>` - `\R` matches CR, LF, or CRLF
  - `<bsr_unicode>` - `\R` matches any unicode newline
  - `<js>` - Javascript compatibility

`Sx` is enabled by default in order to encourage use of whitespace for better
readability.

## Usage

#### `match(string, Regex, start = 0, endpos = -1): RegexMatch`

Tries to match the pattern, starting at start. This means that
`"foo".match(re"f") == true`, but `"foo".match(re"o") == false`.

 - `start`
   - The start point at which to start matching. `|abc` is `0`; `a|bc` is `1`
 - `endpos`
   - The maximum index for a match; `-1` means the end of the string, otherwise
     it's an exclusive upper bound.

[proc-match]: #matchstring-regex-start--0-endpos---1-regexmatch

#### `find(string, Regex, start = 0, endpos = -1): RegexMatch`

Finds the given pattern in the string. Bounds work the same as for
[`match(...)`][proc-match], but instead of being anchored to the start of the
string, it can match at any point between `start` and `endpos`.

[proc-find]: #findstring-regex-start--0-endpos---1-regexmatch

#### `findIter(string, Regex, start = 0, endpos = -1): RegexMatch`

Works the same as [`find(...)`][proc-find], but finds every non-overlapping
match. `"2222".find(re"22")` is `"22", "22"`, not `"22", "22", "22"`.

Arguments are the same as [`match(...)`][proc-match]

Variants:
 - `findAll(...)` returns a `seq[RegexMatch]`
 - `findAllStr(...)` returns a `seq[string]`

[iter-find]: #finditerstring-regex-start--0-endpos---1-regexmatch

#### `split(string, Regex): seq[string]`

Splits the string with the given regex. This works according to the rules that
Perl and Javascript use.

  - If the match is zero-width, then the string is still split:
    `"123".split(r"") == @["1", "2", "3"]`.
  - If the pattern has a capture in it, it is added after the string split:
    `"12".split(re"(\d)") == @["", "1", "", "2", ""]`.

[proc-split]: #splitstring-regex-seqstring

#### `replace(string, Regex, sub): string`

Replaces each match of Regex in the string with `sub`. If `sub` is a `proc
(RegexMatch): string`, then it is executed with each match and the return value
is the replacement value.

If `sub` is a string, then each match is replaced with that string, where the
captures are accessable as `$1`, `$2`, and so on. A literal `$` can be added by
doubling up like so: `$$`.

[proc-replace]: #replacestring-regex-sub-string

### `RegexMatch`

Represents the result of an execution. On failure, it is `nil`. The available
fields are as follows:

 - `pattern: Regex` - the pattern that is being matched
 - `str: string` - the string that was matched against
 - `captures[int|string]: string` - the string value of whatever was captured
   at that id. If the value is invalid, then behavior is undefined. If the id
   is `-1`, then the whole match is returned. If the given capture was not
   matched, `nil` is returned.
 - `captureBounds[int|string]: Option[Slice[int]]` - gets the bounds of the
   given capture according to the same rules as the above. If the capture is
   not filled, then `None` is returned. The upper bound is exclusive, the lower
   bound is inclusive.
 - `match: string` - the full text of the match.
 - `matchBounds: Slice[int]` - the bounds of the match, as in `captureBounds[]`
 - `(captureBounds|captures).asTable` - returns a table with each named capture
   as a key.
 - `(captureBounds|capture).toSeq` - returns all the captures by their number.

### `Pattern`

Represents the pattern that things are matched against, constructed with
`initRegex(string)` or `re(string)`.

 - `pattern: string` - the string that was used to create the pattern.
 - `captureCount: int` - the number of captures that the pattern has.
 - `captureNameId: Table[string, int]` - a table from the capture names to
   their numeric id.
