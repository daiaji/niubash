# Observed Shell-Syntax Quirks vs Real Bash (Session Log)

> First-hand quirks observed while running real-world scripts under niubash,
> recorded per `AGENTS.md` ("fix rubash upstream instead of carrying
> long-term host-side workarounds"). Each entry lists the environment, the
> exact repro, observed behavior, and the current workaround. Treat this as
> an upstream-tracking log: when rubash fixes an entry, move it to the
> "Resolved" section with the fixing rubash version.

- **Shell under test:** niubash 1.1.4 (`bash --version` → "Niubash 1.1.4 —
  bash-compatible shell for Windows"), observed 2026-09-22. Second session
  2026-09-23 (long-running peshell T1-T8 + test-system campaign: ~200 shell
  commands, batch scripts, cmd/PowerShell boundary work) produced Q4-Q12.
- **Reference shell:** GNU bash (POSIX). Constructs below are accepted by
  real bash and by `bash -n` under niubash, but behave differently at
  niubash **execution** time.

---

## Q1. Function definition with a same-line comment after `{` fails to execute

**Repro** (`gate.sh`, reduced):

```bash
FAIL=""
step() { # step <name> <command...>
  echo ok
}
step a
```

**Observed (niubash 1.1.4, execution):**

```
line 2: syntax error near unexpected token `('
line 2: `step ( ) { # step <name> <command...>'
```

- Real bash: runs fine (POSIX allows a comment anywhere a token can start,
  including immediately after `{`).
- niubash `bash -n` on the **same file**: passes. Only **execution** fails —
  so `-n` and the executor evidently use different parser paths, and `-n`
  cannot be trusted to predict executor acceptance for this construct.
- Line endings ruled out: the failing file was verified LF-only by byte
  inspection (`od -c`), and the error persisted after normalization.
- The error rendering also mangles the source line (inserts spaces around
  `()`), so the message does not show the literal input text.

**Workaround:** put the comment on its own line:

```bash
# step <name> <command...>
step() {
  echo ok
}
```

**Upstream candidate:** rubash lexer — allow a comment to start right after
`{` in a brace group; and reconcile `bash -n` with the executor's parser.

---

## Q2. Merged `2>&1 | tail` streams reorder: stderr can appear after buffered stdout

**Repro:** `cargo test --workspace 2>&1 | tail -6` on a crate whose tests
also spawn subprocesses that inherit the pipe (e.g. tests invoking
`ping.exe`).

**Observed:** `tail -6` can show cargo's stderr status lines
(`Finished ...`, `Running unittests ...`) and the child processes' output as
the **last** lines of the merged stream even though the run completed
successfully and printed `test result: ok ...` — the child's unbuffered
writes and the parent's line-buffered stderr land in the pipe **after** the
test harness's block-buffered stdout is flushed. Diagnosing "the run
crashed" from the tail of a merged `2>&1` stream is therefore unreliable.

- This is pipe-buffer interleaving, not a parser defect; real bash behaves
  the same at the OS level. It is recorded here because it produced a false
  "heap corruption crash" diagnosis in a real session.
- niubash-relevant angle: `bash -n`-style trust heuristics do not exist for
  stream order, and agents routinely diagnose from `tail`. A documented
  "don't diagnose from merged-stream tails; separate streams to files" rule
  prevents repeat mistakes.

**Workaround:** redirect stdout and stderr to separate files and inspect
each (`cmd > out.txt 2> err.txt`), or check the exit code
(`echo $?`) instead of the tail shape.

---

## Q3. `exit 0` inside a background loop guarded by `&&` does not terminate the job

**Repro:** a backgrounded probe script of the form

```bash
while true; do
  [ ! -f /tmp/flag ] && echo waiting && exit 0
  # ... work ...
  sleep 2
done
```

run via a background-job wrapper.

**Observed (niubash 1.1.4, background job):** the `exit 0` did not end the
job — the loop kept producing output and the job stayed `running` after the
guard should have exited it.

- Real bash: `exit` terminates the (sub)shell running the loop.
- Suspected area: background-job wrapper treating the loop body's `exit` as
  applying to an inner scope, or the `&&` chain swallowing the exit status —
  not further diagnosed; recorded from one reproducible session observation.

**Workaround:** structure background waits as bounded loops with `break`
plus an exit after the loop, or poll with short-lived foreground runs
instead of a persistent background loop.

---

## Q4. sed `\r` handling is asymmetric: the strip branch no-ops while the add branch works (CR accumulation)

**Repro:** on a CRLF file, repeatedly running the intended CRLF normalization

```bash
sed -i 's/\r//g; s/$/\r/' file.cmd
```

**Observed (niubash 1.1.4, 2026-09-23):** the file accumulated CRs —
`od -c` showed `\r \r \r \n` after three invocations. The `s/$/\r/`
branch inserts a real CR every run, but the `s/\r//g` strip branch does
**not** remove existing CRs. On an LF-only input the same command produced
a correct single CRLF, isolating the defect to the strip branch. A second
session command that used `s/\r//g` to clean CRs before re-adding also
left CRs behind (byte-verified `\r \r \n`).

- Real bash + GNU sed: both branches treat `\r` as CR; the pair converges
  to a single CRLF.
- Consequence: batch CRLF normalization via sed is unreliable and silently
  corrupts batch scripts with stacked CRs.

**Workaround:** `unix2dos` (verified byte-exact), or
`awk '{sub(/\r$/, ""); printf "%s\r\n", $0}'` (verified byte-exact), or
`perl -pi -e` for in-place edits (verified letter-preserving). Never stack
sed CR passes.

**Upstream candidate:** rubash/winuxcmd sed — make `\r` in the pattern
position match CR consistently with the replacement position.

---

## Q5. cmd → `powershell.exe -File`: a trailing backslash inside a quoted argument escapes the closing quote and silently drops all following parameters

**Repro** (batch script `run-sandbox.cmd`):

```bat
set "HERE=%~dp0"
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "%HERE%gen-wsb.ps1" -ScriptDir "%HERE%" -ResultsDir "%RESULTS%" -PayloadDir "%PAYLOAD%" -OutWsb "%WSB%"
```

with `%HERE%` = `D:\repo\peshell\scripts\sandbox\` (trailing `\`).

**Observed:** PowerShell failed with
`Cannot process command because of one or more missing mandatory
parameters: ResultsDir PayloadDir OutWsb` — i.e. the trailing `\"` in
`"...sandbox\""` escapes the quote at the cmd→PowerShell argument
boundary, `-ScriptDir` swallowed the rest, and every later parameter
vanished. Minimal repro with literal paths and no variables succeeded,
isolating the trigger to the trailing backslash inside the quoted arg.

- This is the classic Windows argument-quoting rule (backslash before a
  quote = escape) biting exactly at the cmd→PowerShell handoff; real bash
  → bash has no such layer.
- Diagnosis cost: required a 5-step variable/variable-free bisection
  before the trigger was isolated.

**Workaround:** strip the trailing backslash before use
(`set "PSHERE=%HERE:~0,-1%"`) and pass that to PowerShell; never end a
quoted argument with `\` when the callee is PowerShell.

---

## Q6. `timeout` in a cmd script spawned from niubash resolves to coreutils timeout (PATH shadowing), and Windows `timeout.exe` refuses redirected stdin

**Repro:** `cmd.exe /c script.cmd` where the batch uses
`timeout /t 3 /nobreak >nul` for a poll delay.

**Observed (two stacked failures):**

1. `timeout: invalid time interval '/t'` — the inherited PATH resolved
   `timeout` to Git/coreutils `timeout` instead of
   `%SystemRoot%\System32\timeout.exe` (niubash environment PATH puts the
   Unix tools first; cmd.exe inherits it).
2. After switching to the absolute path:
   `ERROR: Input redirection is not supported, exiting the process
   immediately.` — Windows `timeout.exe` hard-fails when stdin is not a
   console (the script was invoked with `< /dev/null`, standard for
   unattended runs).

- Real bash + GNU coreutils `timeout` would accept the first form; and
  the Windows tool's stdin refusal makes it unusable for unattended
  batch work regardless.

**Workaround:** `ping -n 4 127.0.0.1 >nul` for second-scale delays in
unattended batch scripts (verified: stdin-redirect-safe, no PATH
ambiguity). General rule: in cmd scripts that may run under a
Unix-tools-first PATH, call Windows built-ins by absolute path.

---

## Q7. Win32 `/switch` arguments are mangled under bash: `whoami /priv` prints only the username

**Repro:** `whoami /priv` under niubash → prints `Administrator` and
nothing else (the `/priv` switch never reaches the tool); `cmd //c
"whoami /priv"` → same.

**Observed:** the privileged-information listing only appears when
calling the binary by full Windows path from PowerShell:

```powershell
& "$env:SystemRoot\System32\whoami.exe" /priv
```

- Suspect: slash-argument → path translation at the bash→process
  boundary (same family as the historic `tasklist //FI` double-slash
  workaround, which did work).
- Impact: privilege diagnostics silently degrade to a username print —
  a wrong-but-plausible output that can mislead an agent into
  misdiagnosing elevation state (this session burned a diagnosis cycle
  on exactly that).

**Workaround:** full Windows path via PowerShell for `/switch` tools, or
redirect the tool's output to a file from an elevated cmd.

**Upstream candidate:** rubash/winuxcmd argument translation — pass
`/switch` tokens through verbatim when they are not valid paths.

---

## Q8. Inline `powershell -replace` through bash double quotes rewrote the file with every literal `r` removed

**Repro:** normalizing CRLF via a bash-escaped inline command:

```bash
powershell.exe -NoProfile -Command "\$t = [IO.File]::ReadAllText(\$p) -replace [string][char]13, ''; [IO.File]::WriteAllText(...)"
```

**Observed:** after this ran, the target batch file had **every letter
`r` removed** (`run` → `un`, `runner` → `unne`, `harness` → `haness`;
od-verified). Pattern/intent was CR removal; the executed pattern
evidently degraded to literal `r` somewhere in the
bash → cmd → PowerShell quoting layers. Reproduced twice on two files
before being caught by inspection.

- Real bash + single-layer PowerShell: the CR pattern would be a CR.
- The exact degrading layer was not fully diagnosed (multi-layer escape
  handoff); recorded because silent letter-eating file corruption is the
  most damaging quirk class seen in either session.

**Workaround:** never rewrite files through inline cross-shell
`-replace`. Use `unix2dos`/`awk`/`perl -pi -e` (all letter-safe, verified
byte-exact), or write a `.ps1` file and run it via
`powershell.exe -File`.

---

## Q9. Transient relative-path ENOENT on files that exist (same command, retried, succeeds)

**Repro (observed 3× across the session):**

- `cat scripts/host-elev/t3_disk_out.txt` → `No such file or directory`
  while an immediately preceding `ls scripts/host-elev/` listed the file
  and an earlier identical `cat` had succeeded.
- `grep -rn "TIER: T3" crates/` → `grep: cannot open 'crates/'` while
  `pwd` was correct and the next command with the same relative path
  worked.

**Observed:** transient, self-healing, non-deterministic (roughly 1 in
50-100 relative-path accesses late in long sessions).

- Real bash: not reproducible for an existing path.

**Workaround:** absolute paths for critical reads/aggregations (adopted
as a session rule after the second occurrence); retry once on ENOENT
before believing it.

**Upstream candidate:** rubash cwd/FS-handle state — investigate whether
a cached handle or cwd translation goes stale after external processes
(cmd/PowerShell) change directories.

---

## Q10. `cmd //c "script.cmd"` intermittently prints only the banner and does not run the script

**Repro:** `cmd //c "scripts\\sandbox\\run-sandbox.cmd"` (several
invocations across the session).

**Observed:** some invocations returned only the Windows version banner
+ prompt line with exit 0 — the batch script demonstrably did not
execute (its side effects were absent). Identical invocations at other
times ran fine. A variant with the full absolute path and no `\\`
escaping printed `'D:\...\script.cmd' is not recognized as an internal
or external command` once while working the next time.

- Real bash spawning cmd: deterministic.
- Suspect: argument re-quoting across the niubash → cmd boundary is
  flaky for backslash paths; attached stdin (the pipeline) may also
  matter.

**Workaround (deterministic across ~10 runs):**
`cmd.exe /c "D:\abs\path\script.cmd" > out.txt 2>&1 < /dev/null` —
absolute path, file redirects, stdin detached. Or route through a
`.ps1` file via `powershell.exe -File` (with the Q5 no-trailing-
backslash rule).

---

## Q11. `grep` `\x` escapes need `-P`: the default mode treats them literally, producing misleading counts

**Repro:** counting non-ASCII lines with
`grep -c "[^\x00-\x7F]" file.cmd` returned the **total** line count
(every line matched — the class degraded to "not x / not 0-7 / not F");
`grep -cP "[^\x00-\x7F]"` returned the correct 0.

- Standard GNU grep behavior (PCRE required), but the wrong count nearly
  shipped in an ASCII-only verification step; recorded so agents reach
  for `-P` first.

**Workaround:** `grep -P` for `\x`/`\d` classes; verify a known-good
file alongside when a count gates a decision.

---

## Q12. Background-job wrapper: first `head -N` of streamed output can be lost when the job finishes between wrapper and reader

**Repro:** `bash script.sh > /tmp/x.log 2>&1` as a background job;
immediately reading the job's output after the finish notification.

**Observed:** one run returned only the wrapper's status line without
the script's captured output, while `/tmp/x.log` held the full output.
Not reproduced deterministically; file-first discipline avoids it.

**Workaround:** background jobs should always write to an explicit file;
read the file, treat the wrapper stream as advisory. (Already the
session convention; recorded from one occurrence.)

**Attribution note:** this and Q8's "pwsh tool wrapper ParserError on
scripts containing `$`/quotes" (every `pwsh`-tool call failed with the
wrapper text `...UTF8Encoding]::new($false); "pwsh" -NoLogo -NoProf...`
injected into the parse — workaround: write a `.ps1` file and invoke via
`powershell.exe -File`) originate in the **agent-harness tooling layer**,
not niubash; recorded here for agent awareness only.

---

## Previously known, project-documented in `scripts/refactor-gate.sh` comments (2026-09-22)

- `grep --include`/`--exclude-dir` must precede the search path — after
  it they are treated as file names and grep exits 2 (the gate's
  safety-sentinel silently no-op'd on this).
- `$( ... | grep -v ... )` command substitution captures empty in this
  implementation — use a temp file as the intermediate carrier.

---

## Q13. `vswhere -latest` returns zero instances despite VS2022 with full VC tools installed

**Repro:** `vswhere.exe -latest -products '*' -property installationPath`
(and the `-requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64`
variant) on a host where
`C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.44...\bin\Hostx64\x64\cl.exe`
exists and `rustc` x86_64-pc-windows-msvc builds link fine.

**Observed:** vswhere prints only its banner — zero instance rows, so any
build script that locates `vcvars64.bat` via vswhere concludes "no MSVC"
and falls through to MinGW/toolchain-missing, even though cl.exe is
installed and usable.

**Workaround:** after the vswhere probes, glob the standard installation
roots directly and probe for the batch file:
`/c/Program Files/Microsoft Visual Studio/2022/{Community,Professional,Enterprise,BuildTools}/VC/Auxiliary/Build/vcvars64.bat`
(implemented in peshell `scripts/peer/serial_cmd/build.sh`). Root cause
(Instance registry not readable to vswhere) is a host/VS-installer issue,
not niubash — recorded as a toolchain quirk per the logging mandate.

---

## Q14. `wc -c < file` reports 0 for a freshly written file

**Repro:** `build.sh` prints `serial_cmd.exe（$(wc -c < "$OUT") bytes）`
right after the linker writes the exe; niubash reported `0 bytes` while
`ls -la` showed 283648.

**Observed:** the redirected-stdin form of `wc -c` read zero bytes at
least for the just-created output file; `ls -la` is correct. (Q12-family:
metadata/read path races around freshly produced files.)

**Workaround:** use `ls -la` / `stat -c %s` (or `wc -c` on a path
argument rather than redirected stdin) when the number gates a decision.

---

## Q10 addendum (root cause confirmed deterministic): `cmd //c` under niubash is a silent-success trap

Q10 recorded intermittent behavior. The 2026-09-23 serial_cmd build gave
a deterministic reproduction with a precise mechanism: niubash performs
**no MSYS-style path/argument translation**, so `cmd //c script.cmd`
hands cmd the literal token `//c`; cmd does not recognize it as the
`/c` switch and starts **interactive** with the script name as its first
prompt input — it reads the attached pipeline/stdin, hits EOF, and exits
0 having never executed the batch (banner + prompt in the output, side
effects absent). Hence: with attached non-empty stdin it can *sometimes*
consume and run lines (the "intermittent" face of Q10); with EOF stdin
it is a deterministic silent no-op. The single-slash form
`cmd /c script.cmd` is the correct invocation under niubash (verified:
batch executes every time). If a script must stay Git-Bash-compatible,
normalize inside the script rather than relying on `//c`.

---

## Q15. POSIX path args to child bash scripts arrive Windows-translated: in-script globs on `$1` silently fail

**Repro:** `bash scripts/peer/collect_pe_runtime_dlls.sh /tmp/x` (peshell
toolchain, 2026-09-23). The callee echoed its `$1` as
`C:\Users\...\Temp\x`.

**Observed:** inside the callee, `cp "$SYS32/$f" "$DEST/"` and `find
"$DEST" -type f` (native tools) worked fine — 31 DLLs actually landed —
but `ls "$DEST"/*.dll` produced nothing (bash glob cannot expand a
backslash path), so `ls | wc -l` counted 0 and the script declared
"UCRT forwarders insufficient (0 < 10)" and exited 1 despite a complete
collection. Same pipeline executed interactively in the parent shell on
the POSIX form works (31). I.e. the translation happens **at the
bash→bash invocation boundary only**, and native utilities tolerate the
Windows form while bash globbing does not — a false-failure that looks
like a script bug.

**Workaround:** normalize at script entry —
`case "$1" in [A-Za-z]:\\*) DEST="$(cygpath -u "$1")";; esac`
(cygpath is present under niubash; a sed fallback
`s#^\([A-Za-z]\):#/\L\1#; s#\\#/#g` suffices). Applied in peshell
`scripts/peer/collect_pe_runtime_dlls.sh`. General rule for toolchain
scripts: never glob a path received as an argument without normalizing
it first.

---

## Verified compatible in the same session

For calibration, the following worked as expected under niubash 1.1.4 in
the session that produced Q1/Q2: `mktemp -d`, `cmp -s`, `tr -d '\r'`,
`od -c`, `file`, `tee`, `grep -rln/-rn/-q` with `--include`, `sed -n`
(viewing), `tasklist //FI`, `${PIPESTATUS[0]}`, `$(...)` nesting,
`for`/`if`/`case` without same-line braces comments, `"$@"` in functions
(after the Q1 workaround), `mv` over an existing file, `wc -l`,
`sort | uniq -c`.

Second session (2026-09-23) additions, verified in real campaign use:
`unix2dos`, `perl -pi -e` (letter-preserving in-place edit, unlike sed
Q4 / inline PowerShell Q8), `awk '{sub(/\r$/, ""); printf "%s\r\n", $0}'`
(byte-exact CRLF conversion), `findstr /C:"..."` with multiple patterns,
`taskkill /IM ... /F`, `start ""` non-blocking, `reg query`,
`sc query`, `cmd.exe /c "abs\path.cmd" > file 2>&1 < /dev/null` (Q10
workaround), `ping -n N 127.0.0.1 >nul` as the unattended delay (Q6
workaround), `Get-Disk`/`Get-WindowsOptionalFeature` via
`powershell.exe -NoProfile -Command 'single-quoted'` (Q8 workaround for
the command form; Q7 full-path rule still applies to `/switch` tools),
`grep -P` for `\x` classes (Q11), background jobs writing explicit files
(Q12 workaround), `git -C <repo>`, `cargo build/test/clippy`, `node`,
and long single-quoted `-Command` strings without embedded `$`.
