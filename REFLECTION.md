## 1

The decision is made at `src/scanner.rs:148`, where my scanner checks whether `peek()` is `.` and whether `peek_next()` is a digit before treating the `.` as part of a number.

For the input `5.`, `number` first consumes the digit `5`. At that point, `peek()` returns `.`, while `peek_next()` returns a newline when the file has a trailing newline, or `\0` if the file ends immediately after the dot. Therefore, the condition on line 148 is false. This means the dot is left alone, and the token emitted is `NUMBER '5'`. The `.` is then handled by the next call to `scan_token()`, which reports `Character is not part of any token.`

The scanner therefore reports the error to standard error with exit code 65, and no tokens are printed because `main.rs` stops when there are scan errors.

Section 1.4 requires this because a decimal point can only be part of a number when it is immediately followed by a digit. Therefore, `5.` is not accepted as a single number. If the scanner consumed the dot before checking for a following digit, it would incorrectly produce `NUMBER '5.'` and accept an input that Section 1.4 says is not a valid number.

## 2

My scanner changes the line counter in three places. In `src/scanner.rs:101`, the newline case in `scan_token` increments `self.line` whenever a normal newline is encountered. In `src/scanner.rs:128`, the newline check inside `string` also increments `self.line` so that multi-line strings are tracked correctly. In contrast, `src/scanner.rs:37` does not count a newline; it overrides `self.line` with `eof_line` after scanning has finished.

For the test input containing `print 1;` followed by two blank lines (three newline characters in total), `self.line` is 4 when the main loop finishes. However, the EOF token carries line 1, because line 37 sets `self.line` to the line of the last token before creating EOF.

Section 6.1 requires this so that the token stream does not change depending on how much whitespace or how many comments appear after the actual code. In particular, trailing `// expect:` comments should not move EOF away from the line where the program's final token occurs. For an empty or comments-only file, `unwrap_or(1)` at `src/scanner.rs:34` ensures that EOF uses line 1.

## 3

The test I failed was `tests/phase-1/valid/eof_line.kobo`; it expected `[line 1] EOF ''` but my scanner printed `[line 9] EOF ''`. Nine other valid tests failed the same way.

In commit `7458d41`, the line was still wrong. `src/scanner.rs:35` read:

    self.add(TokenType::Eof);

When I wrote it, I assumed self.line would still hold the line of the last real token when the loop finished, so EOF would land on the line where the code ends. What I had misunderstood was the difference between where the scanner is when it finishes (the running line counter) and which line the last token is on. Section 6.1 says that EOF carries the line of the last real token, not the line the scanner has counted up to by the end of the input. My scanner printed the line number of the end of the file, because every newline in the `// expect:` comments at the bottom of the file incremented self.line even though those comments produce no tokens. In `eof_line.kobo`, all the tokens are on line 1, but the counter reached 9.

I fixed it in commit `4de28f0`. There, `src/scanner.rs:34`, `:37` and `:38` read:

    let eof_line = self.tokens.last().map(|t| t.line).unwrap_or(1);
    self.line = eof_line;
    self.add(TokenType::Eof);

In my final code the fix is at `src/scanner.rs:34` and `src/scanner.rs:37`. It works because line 34 reads the line recorded on the last token already pushed (falling back to 1 if there are none), which ignores whatever the counter did afterward. Line 37 then sets self.line to that value, so add() stamps EOF with the last token's line.
