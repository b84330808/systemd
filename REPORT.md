# systemd fix report

- Upstream: https://github.com/systemd/systemd (default branch `main`)
- Base commit for every branch: `696175871c` ("ci: give the Claude reviewers the UAPI specifications")
- Fork: https://github.com/b84330808/systemd
- Nothing was posted to upstream (no PRs, issues, comments or pushes). Upstream was only read
  (git fetch, GitHub search).

## Fix branches

| # | Branch | Head SHA | Proposed PR title |
|---|--------|----------|-------------------|
| 1 | `fix/percent-util-negative-fraction` | `4ebf9e789485ca263d58131a3fc4f605648f339c` | percent-util: refuse negative values with a zero integer part |
| 2 | `fix/time-util-timestamp-size-check` | `48ba27c32aaea5842c2f3645561f2582607817f5` | time-util: fix operator precedence in format_timestamp_style() size check |
| 3 | `fix/time-util-epoch-timestamp-range` | `9358d3a6dd8a829ca85758f052ba8b3b2b728402` | time-util: apply the timestamp range check to "@" timestamps, too |
| 4 | `fix/env-util-extended-expansion-nest` | `16e0d8c139ef8c50f0b0609bce002a7f34c6baed` | env-util: fix nesting level after unsupported "${FOO:" syntax |

Private finding, see local follow-up: hexdecoct

Every fix branch starts with a commit `README: add review reminder required by AGENTS.md`,
which prepends two lines to `README.md`. Upstream's `AGENTS.md` makes this a hard rule for AI
agents, and says only the human author may remove them, as confirmation that they reviewed the
change. Drop that commit after reviewing, before opening the PR. It is always the first commit
after the base, so for each branch:

```bash
git rebase --onto 696175871c <README-commit-sha> <branch>
```

(The README commit SHA of each branch is listed in its section below.)

## Policy notes (upstream, as of base commit)

- `docs/CONTRIBUTING.md`, "Policy on the use of LLMs and AI tooling": AI-assisted contributions
  are **allowed**, if the human reviews and edits the result. The human is the author and must be
  able to explain every line. AI must not be listed as author or co-author or credited in a way
  that suggests ownership. AI must **not** open PRs, post comments or issues, or write to the
  mailing list on the contributor's behalf: the PR must be opened by the human, and the PR
  description should be reviewed and edited so it reads like existing PRs (short, no "AI voice",
  no headings/bullets for a simple change, no file-by-file summaries). First violation: warning;
  second: ban.
- No CLA, no DCO. `docs/CODING_STYLE.md`: "Do not use Signed-off-by: in your commit messages."
  Commits here therefore have no `Signed-off-by` and no AI trailer.
- Commit subjects use a `component: ` prefix (`docs/CODING_STYLE.md`).
- No PR template in the repository (`.github/` only has issue templates).
- PRs must be against a recent `main`; the test suite must pass; updates are force-pushed to the
  same branch with a comment explaining the change.
- `AGENTS.md` (also `CLAUDE.md` symlink) contains instructions aimed at AI agents:
  - the README.md review-reminder "HARD RULE" described above (followed, as a separate commit);
  - "Only human beings can ever be credited within commit messages" (followed);
  - build/test rules: use `meson test -C build -v <test>` directly, run clang-tidy tests if a
    compatible clang-tidy is available.
- Commits are authored `b84330808 <b84330808@gmail.com>`.
- Upstream CI runs Claude-based reviewers on PRs (base commit: "ci: give the Claude reviewers the
  UAPI specifications"); expect automated review comments in addition to maintainer review.
- Before submitting, upstream expects the test suite to be run locally (`meson test -C build`);
  only the targeted tests listed below were run here, not the full suite.

## Environment

Ubuntu 24.04 container, gcc 13.3, meson 1.9.2 (pip), plus `gperf` and `libcap-dev` from apt.

```bash
meson setup build -Dmode=developer -Dtests=true -Dslow-tests=false -Dinstall-tests=false
```

clang-tidy: only clang-tidy 18 is installed. It cannot parse the repository's `.clang-tidy`
(`unknown key 'RemovedArgs'`, then `ExcludeHeaderFilterRegex`), so the `clang-tidy-*` meson tests
"pass" without checking anything, and with those keys removed it reports `misc-include-cleaner`
findings for untouched code too (it ignores `MissingIncludes: 'false'`). Per `AGENTS.md`, clang-tidy
was skipped rather than installing a newer version. CI will run it.

---

## Fix 1: percent-util: refuse negative values with a zero integer part

Branch `fix/percent-util-negative-fraction`, head `4ebf9e789485ca263d58131a3fc4f605648f339c`.

Commits:

1. `c19fba5afa` README: add review reminder required by AGENTS.md (drop before submitting)
2. `2c85db0e37` percent-util: refuse negative values with a zero integer part
3. `4ebf9e7894` parse-util: refuse negative load averages with a zero integer part

Found by reading the code (no upstream issue). The percent/permille/permyriad parsers in
`src/basic/percent-util.c` parse the integer part with `safe_atoi()` and refuse `v < 0`.
`safe_atoi("-0")` returns 0, so a negative value with a zero integer part passes, and the sign is
lost when the fractional part is added: `parse_permyriad("-0.5%")` returns 50 (= 0.5%), while
`"-7%"` is refused with `ERANGE`. Users of these parsers include `CPUQuota=` (via
`parse_permyriad_unbounded()`), `ManagedOOMMemoryPressureLimit=`, oomd's `SwapUsedLimit=`,
`MemoryPressureAbove=`, `homectl`, `pam_systemd`, `tc` settings in networkd, conditions, and
`systemd-run -p`/`systemctl set-property` (`bus-unit-util.c`). So `CPUQuota=-0.5%` was silently
applied as a 0.5% quota instead of being rejected. The same `"-0"` hole exists in
`parse_loadavg_fixed_point()` in `src/basic/parse-util.c` (`"-0.50"` accepted as 0.50, while
`"-4000.5"` is refused); it is fixed in a second commit. I grepped `src/basic`, `src/shared` and
`src/fundamental` for other integer-part/fraction parsers (`strchr/memchr(..., '.')`); these are
the only ones with this pattern (`parse_size()`, `parse_time()` and `parse_nsec()` already refuse a
leading `-`).

Behaviour change worth mentioning in review: `"-0%"`, `"-0‰"`, `"-0‱"` are now refused with
`ERANGE` too (they used to return 0), consistent with `parse_time()` refusing `"-0"`.

Not a security issue (no memory safety impact).

### Test commands

```bash
meson test -C build -v test-percent-util test-parse-util
```

Also run for regressions, all OK after the fix: `meson test -C build test-percent-util test-parse-util test-conf-parser test-psi-util`.

### Fail before (fix reverted with `git checkout upstream/main -- src/basic/percent-util.c src/basic/parse-util.c`, tests kept)

```
src/test/test-percent-util.c:20: Assertion failed: Expected "parse_percent("-0%")" to fail with error -34/ERANGE, but it succeeded
src/test/test-parse-util.c:867: Assertion failed: Expected "parse_loadavg_fixed_point("-0.50", &fp)" to fail with error -34/ERANGE, but it succeeded
1/2 test - systemd:test-parse-util   FAIL            0.24s   killed by signal 6 SIGABRT
2/2 test - systemd:test-percent-util FAIL            0.23s   killed by signal 6 SIGABRT
Ok:                0
Fail:              2
```

The test aborts at the first failing assertion, so a scratch program linked against
`build/src/shared/libsystemd-shared-263.so` printed every case (before | after):

```
parse_permille             "-7%"    -> -34 (error) | -34 (error)
parse_permille             "-0.5%"  -> 5           | -34 (error)
parse_permyriad            "-0.5%"  -> 50          | -34 (error)
parse_permyriad            "-0.05%" -> 5           | -34 (error)
parse_permyriad            "-0.5‰"  -> 5           | -34 (error)
parse_permyriad_unbounded  "-0.5%"  -> 50          | -34 (error)   (CPUQuota=)
parse_percent              "-0%"    -> 0           | -34 (error)
```

### Pass after

```
1/2 test - systemd:test-parse-util   OK              0.01s
2/2 test - systemd:test-percent-util OK              0.02s
Ok:                2
Fail:              0
```

### Competing PR / issue search (systemd/systemd, issues and PRs, all states)

Queries: `percent negative`, `parse_permyriad`, `permille negative`, `CPUQuota negative`,
`percent-util`, `negative percentage`, `parse_loadavg_fixed_point`, `loadavg negative`,
`safe_atoi "-0"`, `negative zero percent`. No issue or PR about this bug. Related but different:
#43753 (closed, merged as the overflow-guard fix in `store_loadavg_fixed_point()` already on
`main`), #21218 (closed, 2021, percent→2^32 scale clamping). Open PRs #43819 and #43474 mention
percentages but don't touch `percent-util.c` or `parse-util.c` (checked by fetching their heads).

### PR body draft

> safe_atoi() parses "-0" as 0, so the percent, permille and permyriad parsers did not notice
> negative values whose integer part is zero, and the sign was lost once the fractional part was
> added: parse_permyriad("-0.5%") returned 50, while "-7%" is refused with ERANGE. As a result,
> e.g. CPUQuota=-0.5% was silently applied as 0.5%. Refuse the "-" explicitly.
>
> The second commit fixes the same issue in parse_loadavg_fixed_point().

---

## Fix 2: time-util: fix operator precedence in format_timestamp_style() size check

Branch `fix/time-util-timestamp-size-check`, head `48ba27c32aaea5842c2f3645561f2582607817f5`.

Commits:

1. `00334d609a` README: add review reminder required by AGENTS.md (drop before submitting)
2. `48ba27c32a` time-util: fix operator precedence in format_timestamp_style() size check

Found by reading the code (no upstream issue). The minimum-size check at the top of
`format_timestamp_style()` (`src/basic/time-util.c`) reads

```c
if (l < (size_t) (3 + 1 + 10 + style == TIMESTAMP_DATE ? 0 : (1 + 8 + (us ? 1 + 6 : 0) + 1 + (utc ? 3 : 1)) + 1))
```

`+` binds tighter than `==`, which binds tighter than `?:`, so this is
`(14 + style == TIMESTAMP_DATE) ? 0 : (...) + 1`; the condition is never true and the weekday and
date are never counted. The check requires 12..21 bytes instead of 15..35. For normal timestamps the
later `strftime()` calls still catch a short buffer, but for timestamps past
`USEC_TIMESTAMP_FORMATTABLE_MAX` the `"--- XXXX-XX-XX ..."` placeholder is `strcpy()`ed guarded only
by `assert(l >= strlen(xxx[style]) + 1)`, so a 12..23 byte buffer aborts instead of returning NULL,
and with `NDEBUG` (where `assert()` becomes `__builtin_unreachable()`) it would overflow the
buffer. All in-tree callers pass `FORMAT_TIMESTAMP_MAX` (38) sized buffers and the function is not
public API, so this is latent: not a reachable vulnerability. Fix: add the missing parentheses. I
grepped `src/` (single- and multi-line) for the same `a + b == C ? x : y` pattern; this is the only
instance.

The new test calls `format_timestamp_style()` with every buffer size from 0 up to the result
length for each style, checks it returns NULL and does not write past the given size, and checks
the full-size result.

### Test commands

```bash
meson test -C build -v test-time-util
```

Also run for regressions, all OK after the fix: `meson test -C build test-time-util test-calendarspec test-date`.

### Fail before (fix reverted with `git checkout upstream/main -- src/basic/time-util.c`, test kept)

```
test-time-util: ../src/basic/time-util.c:397: format_timestamp_style: Assertion `l >= strlen(xxx[style]) + 1' failed.
#8  0x00005555555680d7 in test_format_timestamp_short_buffer_one (val=253402214399000001, style=TIMESTAMP_PRETTY, result=0x5555555756b6 "--- XXXX-XX-XX XX:XX:XX") at ../src/test/test-time-util.c:710
        l = 12
1/1 test - systemd:test-time-util FAIL             0.22s   killed by signal 6 SIGABRT
Ok:                0
Fail:              1
```

### Pass after

```
1/1 test - systemd:test-time-util OK               0.02s
Ok:                1
Fail:              0
```

### Competing PR / issue search (systemd/systemd, issues and PRs, all states)

Queries: `format_timestamp_style`, `format_timestamp precedence`, `time-util precedence`,
`TIMESTAMP_DATE length`, `format_timestamp buffer size check`, `XXXX-XX-XX`,
`format_timestamp assertion`. No issue or PR about this bug. Possibly overlapping: open issue
#42441 "never say XXXX-XX-XX XX:XX:XX when talking to humans" asks to change the placeholder text
itself; no PR for it was found. If that lands first, this patch may need a rebase, but the size
check is still needed.

### PR body draft

> The minimum buffer size check in format_timestamp_style() was missing parentheses around the
> ?: expression, so it was parsed as (3 + 1 + 10 + style == TIMESTAMP_DATE) ? 0 : ..., which
> never counted the weekday and date. For timestamps that are too large to format, the
> "--- XXXX-XX-XX ..." placeholder is then copied with only an assert() checking the size, so a
> buffer of 12 to 23 bytes hits the assertion instead of getting NULL back. No caller in the tree
> passes such a short buffer, so this is not triggered in practice.

---

## Fix 3: time-util: apply the timestamp range check to "@" timestamps, too

Branch `fix/time-util-epoch-timestamp-range`, head `9358d3a6dd8a829ca85758f052ba8b3b2b728402`.

Commits:

1. `65af4d4756` README: add review reminder required by AGENTS.md (drop before submitting)
2. `9358d3a6dd` time-util: apply the timestamp range check to "@" timestamps, too

Found by reading the code (no upstream issue). `parse_timestamp()` refuses results after
`USEC_TIMESTAMP_FORMATTABLE_MAX` (9999-12-30 23:59:59 UTC on 64-bit), added in 1bb4b028a3 (2017,
"time-util: refuse formatting/parsing times that we can't store ... so that we never end up
formatting dates we can't parse and vice versa"). But the `@` (seconds since the epoch) branch in
`parse_timestamp_impl()` does `return parse_sec(t + 1, ret);` and skips that check (it predates
it, from 2014). Since `parse_sec()` also accepts `infinity`, `parse_timestamp("@infinity")` returned
`USEC_INFINITY`, and `"@300000000000"` was accepted, while the equivalent `"10000-01-01 UTC"` and
`"+infinity"` are refused. `systemd.time(7)` documents `@` as taking "an integer". Callers that
take the value as-is include `journalctl --since/--until`, `coredumpctl --since/--until`,
`systemd-creds --timestamp/--not-after`, `homectl` timestamp fields, `timedatectl set-time`,
`systemd-analyze timestamp` and networkd's DUID-LLT time; `systemctl --when=` refuses infinity
itself. Fix: route the `@` branch through the same final checks (`goto finish`). `@<seconds>` with
time units (e.g. `@5min`) still works. Fix 2 and fix 3 both touch `src/basic/time-util.c` but in
different functions and don't conflict.

### Reproducer with the real binary (`build/systemd-analyze`)

Before:

```
$ systemd-analyze timestamp @infinity
  Original form: @infinity
Normalized form: -
   UNIX seconds: @18446744073709.551615
       From now: -
$ systemd-analyze timestamp @300000000000
  Original form: @300000000000
Normalized form: --- XXXX-XX-XX XX:XX:XX
   UNIX seconds: @300000000000
       From now: 9449 years 7 months left
$ systemd-analyze timestamp '10000-01-01 UTC'
Failed to parse "10000-01-01 UTC": Invalid argument
$ systemd-analyze timestamp +infinity
Failed to parse "+infinity": Invalid argument
```

After:

```
$ systemd-analyze timestamp @infinity
Failed to parse "@infinity": Invalid argument
$ systemd-analyze timestamp @300000000000
Failed to parse "@300000000000": Invalid argument
$ systemd-analyze timestamp @253402214399
  Original form: @253402214399
Normalized form: Thu 9999-12-30 23:59:59 UTC
   UNIX seconds: @253402214399
       From now: 7972 years 11 months left
$ systemd-analyze timestamp @1395716396
Normalized form: Tue 2014-03-25 02:59:56 UTC
```

### Test commands

```bash
meson test -C build -v test-date
```

New cases in `src/test/test-date.c`: `@infinity` must fail; on 64-bit `@253402214399` must pass
(and round-trip) and `@253402214400` must fail; on 32-bit `@2147397247` / `@2147397248`, matching the
existing `2038-01-18 03:14:07 UTC` boundary tests. Also run for regressions, all OK after the fix:
`meson test -C build test-date test-time-util test-calendarspec`.

### Fail before (fix not applied, tests added)

```
Test: @infinity
"@infinity" → "@18446744073709551615" (unexpected)
src/test/test-date.c:48: Assertion failed: Expected "r < 0", but 0 >= 0
1/1 test - systemd:test-date FAIL            0.22s   killed by signal 6 SIGABRT
Ok:                0
Fail:              1
```

### Pass after

```
Test: @253402214399
"@253402214399" → "Thu 9999-12-30 23:59:59.000000 UTC"
Test: @253402214400
1/3 test - systemd:test-time-util    OK               0.02s
2/3 test - systemd:test-calendarspec OK               0.02s
3/3 test - systemd:test-date         OK               0.01s
Ok:                3
Fail:              0
```

### Competing PR / issue search (systemd/systemd, issues and PRs, all states)

Queries: `parse_timestamp @`, `"@infinity"`, `timestamp epoch @ syntax`, `parse_timestamp infinity`,
`USEC_TIMESTAMP_FORMATTABLE_MAX`. No issue or PR about this. Related, already merged and different:
#6327 (2017, refuse very old dates), #34672 (merged, timezone parsing fixes for `systemd-analyze timestamp`),
#29049 (accept `2012-09-22T16:34[:22]`). Issue #6290 (closed) is about very old `--since` dates.

### PR body draft

> parse_timestamp() refuses timestamps that are too large to be formatted, but the "@" syntax
> returned the result of parse_sec() directly and skipped that check. So "@300000000000" was
> accepted and then displayed as "--- XXXX-XX-XX XX:XX:XX", and "@infinity" was accepted as
> USEC_INFINITY, while "10000-01-01 UTC" and "+infinity" are refused. Let "@" timestamps go
> through the same final checks as the other syntaxes.

---

## Fix 4: env-util: fix nesting level after unsupported "${FOO:" syntax

Branch `fix/env-util-extended-expansion-nest`, head `16e0d8c139ef8c50f0b0609bce002a7f34c6baed`.

Commits:

1. `394ec8c862` README: add review reminder required by AGENTS.md (drop before submitting)
2. `16e0d8c139` env-util: fix nesting level after unsupported "${FOO:" syntax

Found by code reading (no upstream issue). In `replace_env_full()` (`src/basic/env-util.c`), with
`REPLACE_ENV_ALLOW_EXTENDED` (used by `env-file.c`'s `merge_env_file()`, i.e. environment.d via
`systemd-environment-d-generator`), `${` increments `nest`. In the `TEST` state, i.e. after
`${FOO:`, any character other than `-`/`+` is treated as unsupported syntax and falls back to
`WORD`, but `nest` is not decremented. After that, no later `${BAR:-x}`/`${BAR:+x}` in the same
value can get back to `nest == 0`, so it is left unexpanded: `FOO=${FOO:x}${XXX:-baz}` stays as is
instead of becoming `${FOO:x}baz`. Same bug class as the merged #20941 (5ef97a7122, missing
`nest--` after a plain `${FOO}`). Fix: decrement `nest` in that branch. The other `state = WORD`
fallback (`:` without `REPLACE_ENV_ALLOW_EXTENDED`) doesn't need it, since `nest` is only consulted
in the extended-only `DEFAULT_VALUE`/`ALTERNATE_VALUE` states.

### Test commands

```bash
meson test -C build -v test-env-util
```

Also run for regressions, all OK after the fix: `meson test -C build test-env-util test-env-file test-fileio`.

### Fail before (fix not applied, tests added)

```
src/test/test-env-util.c:266: Assertion failed: Expected "z == extended ? "FOO=${FOO:x}baz" : "FOO=${FOO:x}${XXX:-baz}"", got "FOO=${FOO:x}${XXX:-baz} != FOO=${FOO:x}baz"
1/1 test - systemd:test-env-util FAIL            0.19s   killed by signal 6 SIGABRT
Ok:                0
Fail:              1
```

### Pass after

```
1/3 test - systemd:test-env-file OK              0.06s
2/3 test - systemd:test-env-util OK              0.01s
3/3 test - systemd:test-fileio   OK              0.02s
Ok:                3
Fail:              0
```

### Competing PR / issue search (systemd/systemd, issues and PRs, all states)

Queries: `replace_env nest`, `env-util extended`, `environment.d expansion`,
`environment.d default value not expanded`, `environment.d ":-"`, `replace_env empty`. Only related
hit: #20941 (closed, merged 2021, the analogous fix for `${FOO}`). #42424 (closed) only changed
the man page.

### PR body draft

> With REPLACE_ENV_ALLOW_EXTENDED, "${FOO:" followed by something other than "-" or "+" is left
> unexpanded, but the nesting level was not decremented, so any later "${BAR:-...}" or
> "${BAR:+...}" in the same string was left unexpanded too. E.g. "FOO=${FOO:x}${XXX:-baz}" in
> environment.d was not expanded to "${FOO:x}baz". This is the same issue #20941 fixed for
> "${FOO}".

---

## Other leads, not fixed

These came out of a code-reading pass over `src/basic` (env-util, string-util, user-util, uid-range)
and were *not* independently re-verified or fixed here. They're candidates for follow-up PRs:

- `replace_env_full()`: `${FOO:-default}` / `${FOO:+alt}` test whether `FOO` is *set*, not whether
  it is *empty*. With `FOO=` set but empty, `${FOO:-def}` gives `""` and `${FOO:+alt}` gives `alt`.
  `environment.d(5)` says "unless the expansion would be empty" / "would have expanded to a
  non-empty value" (and shell `:-`/`:+` behave that way), while the 2016 commit message b82f58bfe3
  says "if the environment variable isn't already set". Fixing it changes behavior, so it needs a
  maintainer decision (code vs. docs); not fixed here.
- `strv_env_replace_consume()` (env-util.c): "Drop other occurrences" calls
  `strv_env_unset(f + 1, *f)` with the full `NAME=value` string, which only matches exact
  duplicates, so other `NAME=` entries survive. Low reachability.
- `find_line()` / `find_line_after()` (string-util.c): only check the first line that *starts
  with* the needle, so `find_line("foobar\nfoo\n", "foo")` returns NULL. Only caller is repart's
  `--append-fstab=auto`.
- `mangle_gecos()` (user-util.c) doesn't replace 0x7F, which `valid_gecos()` rejects.
- `uid_range_overlaps()` (uid-range.c) treats adjacent ranges as overlapping; no callers.
