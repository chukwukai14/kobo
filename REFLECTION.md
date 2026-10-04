# Reflection

## 1. Fractional numbers

The scanner only treats a `.` as part of a number when it is followed by a digit.
This is handled in `src/scanner.rs:157`, where the condition checks both
`self.peek() == '.'` and `self.peek_next().is_ascii_digit()`. I used this because
the specification says a number can have a fractional part only when the dot is
followed by at least one digit.

This also handles the special case `5.` correctly. The scanner first reads `5` as
a NUMBER, but because there is no digit after the `.`, the dot is not included in
the number — it is then scanned separately on the next call to `scan_token` and
produces the error `Character is not part of any token.` Similarly, `.5` does not
become a number, because the scanner only enters `number` when it first sees a
digit — a leading `.` never triggers that path.

## 2. Line counting and EOF

There are two places where the scanner increments the line counter: `src/scanner.rs:112`
handles a newline encountered during normal scanning, and `src/scanner.rs:135` handles
a newline found inside a string literal. The second is necessary because strings can
span multiple lines (section 1.5), and each line they cross still has to be counted.

For a file ending in two blank lines, the line counter keeps advancing as those blank
lines are scanned, but EOF should not report wherever that counter ends up — those
lines contain no real token. Instead, `run` computes `eof_line` from the last token
already in `self.tokens` (`src/scanner.rs:36-40`), falling back to line 1 if there are
no tokens at all, and uses that value when pushing the EOF token. This matches section
6.1's rule that EOF carries the line of the last real token, not the physical end of
the file.

## 3. Debugging mistake

While reworking `run` to add my `string` and `identifier` implementations, I
accidentally reintroduced a bug I had already fixed earlier. In commit `9cc1cf4`, I
removed the `eof_line` calculation entirely and set the EOF token's line directly
with `line: self.line` in `run`. This broke the EOF line number for any file with
trailing blank lines after the last real token, because `self.line` keeps
incrementing through those blank lines even though no real token exists there — I
had misunderstood this rule once already and reverted the fix without noticing while
editing nearby code.

I fixed it again in commit `cd751ed` by restoring the calculation:

```rust
let eof_line = self
    .tokens
    .last()
    .map(|token| token.line)
    .unwrap_or(1);
```

and using `line: eof_line` in the `Token` push in `run`. This matches section 6.1:
EOF carries the line of the last real token, not wherever the scanner's line counter
has drifted to by the time scanning finishes.