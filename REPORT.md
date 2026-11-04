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
