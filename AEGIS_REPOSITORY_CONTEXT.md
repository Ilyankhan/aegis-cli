# Aegis CLI — Comprehensive Repository Context

> **Purpose of this document:** a research-grade technical context package for NotebookLM, architecture review, onboarding, threat-model analysis, and software-engineering research. It synthesizes the repository's implementation, design documents, build system, tests, security model, operational workflows, and notable inconsistencies.
>
> **Repository:** `Ilyankhan/aegis-cli`  
> **Repository description:** Ultra-fast, zero-dependency DevSecOps CLI scanner written in Go that inspects git diffs and CI pipelines to prevent secret and credential leaks.  
> **Analyzed repository state:** default branch content at commit `b199dd25e44b354fce7574cdb677d18aab86a257`  
> **Primary language:** Go  
> **Declared Go version:** Go 1.23 in `go.mod`  
> **License:** MIT

---

## 1. Executive Summary

Aegis CLI is a local-first DevSecOps command-line security scanner. Its main purpose is to detect accidentally committed credentials, API tokens, private keys, database connection strings, and high-entropy secret assignments before they enter Git history or CI/CD systems.

The tool is designed around five product ideas:

1. **Fast local protection:** scan staged Git changes through a pre-commit hook with minimal developer-visible latency.
2. **Pattern plus entropy detection:** combine provider-specific regular expressions with Shannon entropy and structural heuristics.
3. **Git-aware scanning:** inspect ordinary files, staged diffs, commit-range diffs, and reachable Git history.
4. **Actionable findings:** report severity, confidence, masked values, blast radius, remediation text, and suggested commands.
5. **Optional live verification:** when explicitly enabled with `--verify`, send only structurally recognized provider credentials to read-only provider endpoints.

The executable is implemented almost entirely with the Go standard library. The repository has no third-party Go dependencies in `go.mod`; this gives the binary a small operational footprint and supports static cross-platform builds.

---

## 2. Repository Inventory

### Top-level entries

```text
.aegisignore                 Repository-specific scan exclusions
.github/workflows/           CI and release automation
.gitignore                   Git working-tree exclusions
.golangci.yml                Static-analysis/lint configuration
AEGIS_CLI_SPEC.md            Platform integration and JSON contract specification
LICENSE                      MIT license
Makefile                     Build, test, benchmark, install, and cross-compile commands
PRD.md                       Product requirements and architecture intent
README.md                    User-facing documentation
cmd/aegis/main.go            CLI entrypoint and command orchestration
go.mod                       Go module declaration
install.sh                   Linux/macOS/WSL/Git Bash installer
install.ps1                  Windows PowerShell installer
internal/analyzer/            Detection engine, rules, entropy, and heuristics
internal/config/              Ignore rules, scan settings, cloud configuration
internal/git/                 Git diffs, history scans, and pre-commit hooks
internal/reporter/            Console and JSON reporting
internal/validator/           Provider-specific active credential verifiers
pkg/models/                   Shared public data structures and hashing utilities
```

### File counts by functional area

- **CLI:** `cmd/aegis/main.go`, `cmd/aegis/main_test.go`
- **Analyzer:** engine, entropy, heuristics, rules, and corresponding tests
- **Git integration:** staged/range diff parsing, history traversal, hook management, and tests
- **Configuration:** ignore matching, scan options, cloud login/synchronization
- **Reporting:** ANSI console output, JSON serialization, reporter tests
- **Validation:** provider verifiers and registry tests
- **Shared models:** finding/report types and deterministic hashing
- **Automation:** GitHub Actions CI and release workflows
- **Distribution:** Makefile plus shell and PowerShell installers

---

## 3. Technology and Engineering Stack

### Languages

- **Go:** approximately 95.8% of the repository according to the supplied language composition. Go implements the CLI, scanner, Git integration, reporting, cloud integration, validators, and tests.
- **Shell:** approximately 1.8%. Used by `Makefile` recipes, the Unix installer, and the generated Git hook script.
- **PowerShell:** approximately 1.5%. Used by the Windows installer.
- **Makefile:** approximately 0.9%. Used as the developer-facing build interface.

### Runtime and build model

- Go module: `github.com/Ilyan321/aegis-cli`
- Declared Go version: `1.23`
- Static build mode: `CGO_ENABLED=0`
- Binary optimization: `-ldflags="-s -w"` and `-trimpath`
- Supported release targets: Linux amd64/arm64, macOS amd64/arm64, Windows amd64
- External runtime requirements:
  - Go is required only when installing from source or through `go install`.
  - Git is required for staged scans, range scans, history scans, hook management, and platform metadata discovery.
  - Network access is required only for login, cloud synchronization, or optional active verification.

### Important libraries and standard packages

There are no third-party dependencies declared in `go.mod`. The implementation relies on Go standard-library packages including:

- `regexp`: provider signature matching using Go's RE2-based regular-expression engine.
- `bufio` and `io`: streaming file, diff, and Git object processing.
- `os`, `path/filepath`, and `os/exec`: filesystem operations and Git subprocess integration.
- `crypto/sha256` and `encoding/hex`: deterministic finding and report fingerprints.
- `net/http`, `context`, and `time`: bounded active verification and platform communication.
- `flag`: command-specific command-line parsing.
- `sync` and goroutines: bounded file-scanning concurrency and shared counters.
- `encoding/json`: machine-readable reports and cloud configuration.

---

## 4. Architectural Shape

Aegis is organized as a pipeline:

```text
CLI arguments
    |
    v
cmd/aegis/main.go
    |
    +--> path traversal --------------------+
    |                                        |
    +--> Git staged diff parser              |
    |                                        v
    +--> Git range diff parser        internal/analyzer.Engine
    |                                        |
    +--> Git history / cat-file             +--> prefix gate
                                             +--> regular-expression rules
                                             +--> placeholder filtering
                                             +--> entropy filtering
                                             +--> confidence assessment
                                             +--> deterministic finding model
                                                    |
                          +-------------------------+-------------------------+
                          |                           |                         |
                          v                           v                         v
                  optional validator            console reporter         JSON reporter
                          |                           |                         |
                          +---------------------------+-------------------------+
                                                      |
                                                      v
                                           exit code / optional cloud sync
```

### Package responsibilities

- `cmd/aegis`: user-facing commands, flags, orchestration, exit-code policy, completion scripts, login, and cloud-sync invocation.
- `internal/analyzer`: central detection engine and detection rule registry.
- `internal/config`: ignore matching, scan options, login configuration, platform synchronization.
- `internal/git`: Git subprocess calls, unified diff parsing, history object streaming, and hook lifecycle.
- `internal/reporter`: console rendering and JSON output.
- `internal/validator`: opt-in provider API checks.
- `pkg/models`: shared finding/report contracts, severity/category enums, masking, and hashing.

The `internal` boundary prevents external packages from importing implementation details, while `pkg/models` contains reusable domain structures intended to be shared across the executable and integrations.

---

## 5. CLI Command Surface

The main entrypoint dispatches on the first argument and also supports Git-like convenience behavior: flags can be passed directly, and an existing path can be scanned without explicitly writing `scan`.

### Commands

| Command | Behavior |
|---|---|
| `aegis init` | Installs a pre-commit hook when inside a Git repository and creates a starter `.aegisignore` if missing. |
| `aegis uninit` | Removes the Aegis hook and restores a backed-up previous hook when present. |
| `aegis status` | Displays hook state, ignore-file state, and a quick staged-buffer scan. |
| `aegis scan [path]` | Recursively scans a directory or scans one file. Defaults to `.`. |
| `aegis staged` | Scans added lines in `git diff --cached --no-color -U0`. |
| `aegis scan --range=BASE...HEAD` | Scans added lines in a commit-range diff for CI or pull requests. |
| `aegis audit history` | Delegates to a full reachable Git history scan. |
| `aegis history` | Alias-style route to history scanning. |
| `aegis check "text"` | Scans one directly supplied string. |
| `aegis hook install` | Installs the pre-commit hook explicitly. |
| `aegis hook uninstall` | Removes the Aegis hook explicitly. |
| `aegis completion bash\|zsh\|fish` | Emits shell completion code. |
| `aegis login` | Validates an Aegis Platform token and saves local cloud configuration. |
| `aegis version` | Prints version `1.0.0`. |

### Scan flags

- `--staged`: scan the Git index.
- `--history`: scan reachable Git history.
- `--range=BASE...HEAD`: scan a three-dot Git range.
- `--verify`: enable live provider verification.
- `--format=console|json`: choose human-readable or structured output.
- `--output=PATH`: write JSON to a file; console mode also writes JSON to the requested file.
- `--fail-on=critical|high|medium|low`: define the severity threshold for exit code `1`.
- `--no-color`: suppress ANSI color codes.
- `--sync`: send the report to the Aegis Platform API.

### Argument normalization

The standard Go `flag` package stops parsing at the first positional argument. The `rearrangeArgs` helper moves flags before positional arguments so commands such as `aegis scan . --format=json` work as users expect.

---

## 6. Detection Engine

The analyzer is centered on `internal/analyzer/Engine`.

### File-level safeguards

`ScanFileWithStats` applies these controls before scanning:

- Files larger than **25 MB** are skipped.
- Directories and directory symlinks are not scanned as files.
- Input is streamed through a buffered reader.
- The first **1,024 bytes** are inspected for binary indicators.
- UTF-8 and UTF-16 BOMs are treated as text rather than binary.
- A null byte or more than 25% non-printable control bytes causes a file to be skipped.

### Line-level pipeline

`ScanLine` processes one line in this order:

1. Skip empty or whitespace-only lines.
2. Skip lines containing a recognized inline suppression directive.
3. If the line exceeds **1,200 characters**, use narrow prefix-window sampling rather than applying every rule to the entire minified line.
4. For normal lines, iterate through the default rule registry.
5. Apply a prefix pre-filter when a rule declares prefixes.
6. Run the compiled Go regular expression.
7. Extract the first capture group when available; otherwise use the complete match.
8. Clean surrounding quotes, whitespace, delimiters, and punctuation.
9. Reject obvious placeholders.
10. Apply entropy and low-variance checks when required by the rule.
11. Compute a deterministic finding ID.
12. Assess confidence using line context and file path.
13. Create a `models.Finding` containing masked output, remediation, severity, confidence, and an unverified status.
14. Remove generic duplicate findings when a provider-specific rule already matched the same raw token.

### Long-line strategy

Minified JavaScript and generated assets can contain very long lines. For lines longer than 1,200 characters, `scanLongLinePrefixSampling`:

- searches for known provider prefixes using `strings.Index`;
- extracts a local window of up to 40 characters before and 160 characters after the prefix;
- applies the relevant rule only to that window;
- preserves the original column offset;
- performs the same placeholder, entropy, confidence, and deduplication steps.

This is a performance-oriented compromise: it avoids scanning a huge minified line with every expression while still finding recognizable provider tokens.

### Concurrency

Directory scans collect eligible paths, then create a worker pool sized from `runtime.NumCPU()`, capped by the number of files. Workers scan files concurrently, while a mutex protects shared finding and statistics updates.

The history scan itself streams Git objects sequentially through `git cat-file --batch`; provider verification is rate-limited and currently iterated through findings in order.

---

## 7. Entropy and Heuristics

### Shannon entropy

`CalculateShannonEntropy` implements:

```text
H(X) = -Σ P(xᵢ) log₂(P(xᵢ))
```

It counts rune frequencies, calculates each character's probability, and sums the information contribution. Empty strings return zero.

### Alphabet classification

`DetectAlphabet` categorizes candidate strings as:

- hexadecimal;
- base64-like;
- alphanumeric plus punctuation;
- unknown for empty input.

### Thresholds

| Alphabet | Minimum length | Entropy threshold |
|---|---:|---:|
| Hexadecimal | 32 | 3.0 |
| Base64-like | 20 | 4.5 |
| Alphanumeric/punctuation | 16 | 4.7 |

A special rule rejects 40-character hexadecimal strings below entropy `3.95`, reducing false positives from Git commit SHA values.

### Repetition and sequence filtering

`HasLowVarianceOrSequential` rejects candidates that:

- are shorter than four characters;
- contain fewer than four unique runes;
- consist of a repeated chunk of length 2–10;
- contain a monotonic increasing or decreasing sequence covering more than half of the string.

### Placeholder detection

`IsObviousPlaceholder` rejects common documentation and test values such as `your_...`, `replace_...`, `dummy`, `fake`, `sample`, `example`, `TODO`, and trivial passwords. The standard AWS documentation example key ending in `EXAMPLE` is intentionally treated as an exception and is not rejected by this helper, although test-path confidence may still lower its severity for pipeline blocking.

### Confidence scoring

`AssessConfidence` returns high or low confidence. It lowers confidence when:

- the surrounding line contains mock/test terms;
- the path looks like a test, fixture, mock, specification, or test-data path;
- the token itself resembles a dummy value.

The confidence system is operationally important because low-confidence findings are skipped by the blocking exit-code calculation.

### Suppression directives

The analyzer recognizes text containing:

- `aegis:ignore` and `aegis: ignore`;
- `nolint:aegis`;
- `pragma: allowlist`;
- language-comment forms represented in `IgnoreDirectives`, including `//`, `#`, `/*`, HTML, semicolon, and SQL-style comment prefixes.

The suppression is line-wide: if a line contains an ignore directive, that line is not scanned.

---

## 8. Rule Registry and Supported Credential Classes

Rules are declared in `internal/analyzer/rules.go` as `Rule` values containing an ID, description, category, severity, optional prefixes, compiled regular expression, optional entropy requirements, and remediation metadata.

### Provider-specific and structural rule families

| Rule family | Category | Typical credential type | Default severity |
|---|---|---|---|
| `AEGIS-AWS-001/002` | AWS | Access key ID and secret-access-key assignment | Critical |
| `AEGIS-GH-001/002` | GitHub | Classic and fine-grained personal access tokens | Critical |
| `AEGIS-STRIPE-001/002` | Stripe | Live secret and restricted keys | Critical / High |
| `AEGIS-ANTHROPIC-001` | Anthropic | Claude API key | Critical |
| `AEGIS-DEEPSEEK-001` | DeepSeek | DeepSeek API key | Critical |
| `AEGIS-OPENAI-001` | OpenAI | Project, admin, and legacy-style API keys | Critical |
| `AEGIS-GROK-001` | xAI/Grok | xAI API key | Critical |
| `AEGIS-GROQ-001` | Groq | Groq API key | Critical |
| `AEGIS-PPLX-001` | Perplexity | Perplexity API key | Critical |
| `AEGIS-GEMINI-001` | Gemini | Gemini/AI Studio key in provider context | Critical |
| `AEGIS-GCP-001` | GCP | Google API key | High |
| `AEGIS-HF-001` | Hugging Face | User access token | High |
| `AEGIS-DO-001` | DigitalOcean | Personal access token | Critical |
| `AEGIS-GITLAB-001` | GitLab | Personal access token | Critical |
| `AEGIS-NPM-001` | NPM | Registry access token | Critical |
| `AEGIS-RESEND-001` | Resend | Email API key | Critical |
| `AEGIS-LINEAR-001` | Linear | API key | High |
| `AEGIS-SENTRY-001` | Sentry | Auth token | High |
| `AEGIS-DISCORD-001` | Discord | Bot token | High |
| `AEGIS-TWILIO-001` | Twilio | Account SID | Medium |
| `AEGIS-SENDGRID-001` | SendGrid | API key | Critical |
| `AEGIS-SUPABASE-001` | Supabase | Personal/project token | Critical |
| `AEGIS-SLACK-001` | Slack | Bot token | High |
| `AEGIS-KEY-001` | Generic/private key | RSA, EC, DSA, OpenSSH private key marker | Critical |
| `AEGIS-DB-001` | Database | PostgreSQL, MySQL, MongoDB, or Redis URI with credentials | Critical |
| `AEGIS-GEN-001` | Generic | High-entropy API-key/secret/token assignment | High |

Each rule includes a remediation description. Examples include rotating provider keys, deleting exposed access keys, removing credentials from Git history, changing database passwords, and moving hardcoded secrets into environment variables or a secret manager.

### Matching design

Rules with distinctive prefixes use a cheap `strings.Contains` check before regular-expression evaluation. Rules with no prefix, such as Discord tokens and generic high-entropy assignments, require broader evaluation.

The implementation uses Go's `regexp` package, which is based on RE2 and avoids the catastrophic backtracking behavior associated with many backtracking regex engines. The PRD describes an Aho–Corasick or equivalent prefix gate, but the code implements prefix checks using ordinary string containment rather than a dedicated Aho–Corasick automaton.

---

## 9. Shared Data Model

`pkg/models/finding.go` defines the data contract used by analyzers, validators, reporters, and cloud synchronization.

### `Finding`

Important fields include:

- `ID`: deterministic SHA-256 finding fingerprint.
- `RuleID` and `RuleDescription`: signature identity and human-readable explanation.
- `Category`: provider or credential taxonomy.
- `FilePath`, `LineNumber`, `Column`: source location.
- `MaskedValue`: first four characters retained, remaining characters replaced with asterisks.
- `Entropy` and `EntropyAlphabet`: heuristic metadata.
- `Severity`: critical, high, medium, low, or info.
- `Confidence`: high, medium, or low.
- `Verification`: provider, status, timestamp, details, and latency.
- `Remediation`: blast radius, action, suggested commands, and optional automated diff.
- `Commit`: optional historical commit metadata.
- `RawSecret`: excluded from JSON with `json:"-"`.

### `ScanReport`

The report contains version, target, scan type, timestamp, duration, file and line counts, severity counters, active-leak count, findings, and an order-independent report hash.

### Secret masking

`MaskSecret` preserves only the first four runes. Empty values remain empty, and values of four characters or fewer are completely masked.

### Deterministic finding hash

The finding fingerprint is:

```text
SHA256(normalized_file_path + ":" + line_number + ":" + rule_id + ":" + raw_secret)
```

Paths are cleaned and normalized to slash separators, with a leading `./` removed. This is intended to make fingerprints stable across Windows, macOS, and Linux.

### Deterministic report hash

`ComputeReportHash` sorts all finding IDs, concatenates them without separators, and hashes the result. A clean report returns an empty string. Sorting makes the report hash independent of discovery order.

---

## 10. Git Integration

### Staged diff scanning

`internal/git/diff.go` executes:

```text
git diff --cached --no-color -U0
```

`ParseUnifiedDiff` recognizes:

- `+++` target-file headers;
- `@@` hunk headers and new-line starting positions;
- added lines beginning with `+`;
- context lines beginning with a space;
- deleted files represented by `/dev/null`.

Only added lines are scanned, which is appropriate for pre-commit protection and avoids reporting unchanged legacy code in a developer's current commit.

### Range scanning

Range scanning executes `git diff BASE...HEAD --no-color -U0`, parses the result with the same unified-diff parser, and scans only added lines. This supports pull-request or merge-request security gates.

### History scanning

`internal/git/history.go` performs a repository-wide audit:

1. Run `git rev-list --all --objects` to enumerate reachable objects and paths.
2. Build a SHA-to-path map.
3. Start `git cat-file --batch`.
4. Stream object IDs to Git through stdin.
5. Read object headers and payloads exactly by declared byte size.
6. Inspect only blob objects.
7. Apply ignore rules, file-size limits, binary detection, and line scanning.
8. Resolve commit metadata with `git log --find-object` and cache results.
9. Deduplicate findings by deterministic finding ID.

This is a Git-aware historical audit, but it uses Git subprocesses and the Git object protocol rather than implementing a native parser for pack files. The PRD's wording about direct pack traversal is therefore broader than the actual implementation.

### Hook management

`internal/git/hook.go` manages `.git/hooks/pre-commit` or the repository's configured hooks path.

- Installation is idempotent when the Aegis marker is already present.
- A foreign existing hook is backed up as `pre-commit.backup`.
- Uninstallation restores the backup when available.
- A hook not marked as Aegis is never deleted.
- The generated script invokes `aegis staged`, or `./bin/aegis staged` as a fallback.
- If no binary is available, the hook prints a warning and exits successfully, meaning scanning is fail-open in that specific missing-binary scenario.

---

## 11. Ignore and Configuration System

### `.aegisignore`

`internal/config/ignore.go` loads default ignore patterns and then appends repository-specific patterns from `.aegisignore`.

The loader:

- checks the target directory first;
- walks upward until it finds `.aegisignore` or a `.git` directory;
- ignores blank lines and lines beginning with `#`;
- normalizes slash direction;
- supports exact matches, directory prefixes, basename globs, and normalized-path globs.

### Default exclusions

Defaults include Git metadata, dependency directories, virtual environments, build outputs, lockfiles, binary/media assets, fonts, and some generated artifacts. The repository's checked-in `.aegisignore` additionally excludes test Go files and several operating-system or archive formats.

### Scan options

`ScanOptions` stores target directory, staged/history mode, verification mode, output format, output path, and failure threshold. `NewDefaultScanOptions` uses the current directory, console output, verification disabled, and a critical default threshold.

### Cloud configuration

Cloud credentials are stored in:

```text
~/.aegis/config.json
```

The file is created with directory mode `0700` and file mode `0600`. Environment variables override or supplement file settings:

- `AEGIS_API_URL`
- `AEGIS_API_TOKEN`

The default platform URL is configured as `https://aegis-platform-wwgp.onrender.com`.

---

## 12. Active Verification System

Verification is opt-in. The registry in `internal/validator/verifier.go` creates an HTTP client with:

- a hard **1.5-second** client timeout;
- disabled compression;
- up to ten idle connections;
- a ticker-based interval of approximately **334 ms**, targeting roughly three requests per second.

The registry selects a verifier using both category and token structure. Generic credentials, database connection strings, private keys, and unsupported provider tokens are skipped rather than sent externally.

### Registered verifiers

The registry registers verifiers for Stripe, GitHub, OpenAI, AWS, Hugging Face, GitLab, NPM, DigitalOcean, SendGrid, Supabase, Groq, Resend, and xAI/Grok.

Each verifier follows a similar contract:

1. Confirm category and prefix/length support.
2. Create a read-only HTTP request.
3. Attach the token in an authorization header or provider-specific request format.
4. Interpret HTTP status codes or provider response text.
5. Return active, revoked, error, or skipped status.
6. Record provider, timestamp, details, and latency.

### Representative verification behavior

- **GitHub:** `GET https://api.github.com/user`; `200` means active and includes OAuth scopes when available; `401` means revoked/invalid.
- **Stripe:** `GET https://api.stripe.com/v1/balance`; `200` means active; `401` means invalid/revoked.
- **OpenAI:** `GET https://api.openai.com/v1/models`; `200` means active; `401` means revoked; a `429` containing `insufficient_quota` is treated as evidence that the key is valid but out of quota.
- **AWS:** posts to the STS `GetCallerIdentity` endpoint and interprets `InvalidClientTokenId` as revoked and `SignatureDoesNotMatch` as evidence that the access-key ID is recognized. This is not a complete SigV4 credential validation because the implementation uses a placeholder signature.
- **Groq, Grok, Resend, GitLab, and others:** use provider read-only endpoints and generally classify `200` as active and `401` as revoked.

### Security posture

The central safety principle is structural determinism: only known provider-shaped credentials may be sent to a provider. Generic secrets are explicitly skipped. However, active verification is still a network side effect and should be treated as potentially sensitive operational behavior in privacy-restricted or air-gapped environments.

---

## 13. Reporting and Output

### Console reporter

`PrintConsoleReport` produces colorized terminal output unless `--no-color` is used. It displays:

- pass/fail banner;
- number of secrets;
- rule ID and description;
- file, line, and column;
- masked token;
- entropy and alphabet;
- active/revoked/error verification status;
- historical commit information;
- impact and remediation;
- suggested shell commands;
- scan target and type;
- files/lines scanned;
- latency;
- severity totals;
- active leaks;
- deterministic report hash.

### JSON reporter

`GenerateJSONReport` uses indented standard-library JSON serialization. `WriteJSONReport` either writes to a path or appends a newline to the provided writer. `RawSecret` is excluded by the model's JSON tag, preventing raw credential output in normal JSON reports.

### Report portability

The structured report is intended for CI/CD ingestion, SARIF conversion, platform databases, incident triage, finding deduplication, and compliance workflows. The repository's platform specification recommends indexing on the deterministic finding ID and using the order-independent report hash for idempotency.

---

## 14. Exit-Code Semantics

The main scan flow calculates a blocking count after reporting.

- **Exit `0`:** no findings meeting the configured threshold, or only low-confidence findings.
- **Exit `1`:** one or more high-confidence findings meet or exceed `--fail-on`.
- **Exit `2`:** invalid arguments, missing Git context, unreadable target, failed diff/history operation, or output/configuration errors.

Threshold behavior:

- `LOW`: all non-low-confidence findings block.
- `MEDIUM`/`MED`: critical, high, and medium block.
- `HIGH`: critical and high block.
- `CRITICAL`/`CRIT`: only critical blocks.
- Unknown values fall back to critical behavior.

One important implementation detail: the README describes a low default threshold in one section, while `main.go` and `NewDefaultScanOptions` use `critical` as the default. The actual CLI implementation should be treated as authoritative for current behavior.

---

## 15. Build, Test, and Release Workflows

### Make targets

```text
make                 Build the binary
make build           Build bin/aegis as a stripped static binary
make install         Install to /usr/local/bin or ~/.local/bin
make cross-compile   Produce Linux, macOS, and Windows binaries
make clean           Remove bin/ and dist/
make test            go test -v -race ./...
make bench           Run benchmarks with memory statistics
make fmt             go fmt ./...
make vet             go vet ./...
```

The Makefile declares a `lint` target in the README's example usage, but the checked-in Makefile does not define `lint`; lint configuration exists in `.golangci.yml`, but there is no corresponding Make target or CI invocation.

### Continuous integration

`.github/workflows/ci.yml` runs on pushes and pull requests targeting `main` and performs:

1. checkout;
2. Go 1.23 setup with caching;
3. `go test -v -race ./...`;
4. `make build`;
5. version check;
6. hook install/uninstall integration check.

### Release automation

`.github/workflows/release.yml` triggers on tags matching `v*`. It:

1. runs race-enabled tests;
2. cross-compiles five platform/architecture combinations;
3. injects the tag version into `main.Version`;
4. copies `README.md` and `LICENSE` into each archive directory;
5. produces tarballs for Unix-like systems and a ZIP for Windows;
6. writes `checksums.txt` using SHA-256;
7. publishes a GitHub Release using `softprops/action-gh-release@v2`.

### Installer behavior

`install.sh` detects operating system and architecture. If Go is installed, it prefers `go install`; otherwise, it downloads the latest release archive. It supports Linux, macOS, WSL, Git Bash, and Windows-like shell environments.

`install.ps1` prefers `go install`, otherwise downloads the latest Windows amd64 archive, extracts it under the user's local programs directory, and updates the user PATH.

---

## 16. Test Strategy

The repository uses Go's built-in testing package, integration-style subprocess tests, HTTP test servers, and benchmarks.

### CLI tests

`cmd/aegis/main_test.go` validates:

- version output;
- help output;
- status command behavior;
- clean string checking;
- staged JSON scanning;
- flags after positional paths;
- unknown-command failure;
- Bash, Zsh, and Fish completion output.

### Analyzer tests

Analyzer tests cover:

- provider rule detection across cloud, SaaS, AI, database, and messaging signatures;
- inline suppression;
- binary detection;
- UTF-16 BOM handling;
- long minified-line scanning;
- placeholder filtering;
- test-path recognition;
- confidence downgrading;
- entropy behavior;
- micro-benchmarks for normal and long-line scanning.

### Git tests

Git tests exist for diff parsing, history behavior, and hook installation/uninstallation. The implementation uses real Git commands, so these tests exercise process integration as well as parser logic.

### Validator tests

Validator tests use `httptest.Server` to avoid contacting real provider APIs. They verify:

- generic passwords are skipped;
- active versus revoked provider responses;
- GitHub scope extraction;
- OpenAI insufficient-quota handling;
- AWS XML error interpretation;
- timeout behavior.

### Reporting tests

Reporter tests verify clean console output, finding output, active-leak display, remediation commands, JSON file creation, and JSON deserialization.

### Concurrency and correctness goals

CI runs the race detector. The analyzer protects shared directory-scan counters and finding aggregation with a mutex, while the history scanner protects its commit metadata cache using an RWMutex.

---

## 17. Intended Operational Workflows

### Developer pre-commit workflow

```text
aegis init
    |
    +--> write .aegisignore if absent
    +--> install .git/hooks/pre-commit
    |
    git add ...
    git commit
    |
    +--> hook runs aegis staged
          |
          +--> added lines parsed
          +--> findings rendered
          +--> commit allowed or rejected
```

### CI pull-request workflow

```text
aegis scan --range=origin/main...HEAD --format=json --no-color
```

The scanner evaluates only added lines in the range, returns JSON, and uses exit code `1` when findings meet the configured threshold.

### Historical remediation workflow

```text
aegis audit history
```

This finds credentials that may have been deleted in the working tree but remain present in reachable Git objects. Historical findings can include commit metadata for incident response.

### Platform synchronization workflow

```text
aegis login --token "$AEGIS_TOKEN"
aegis scan --format=json --sync
```

The CLI authenticates against `/api/v1/auth/me`, stores a local token, gathers repository/branch/commit metadata, and posts the report to `/api/v1/scans/cli`.

---

## 18. Security and Privacy Analysis

### Positive security properties

- Raw secrets are masked in console output.
- Raw secrets are excluded from JSON serialization.
- Generic credentials are not sent to external APIs during verification.
- Active verification is explicit rather than default.
- HTTP requests have a hard timeout and a rate limiter.
- Findings and reports are deterministic and deduplicable.
- Existing foreign Git hooks are backed up and protected from accidental deletion.
- Configuration files use restrictive filesystem permissions.
- Binary and oversized files are skipped to reduce attack surface and resource use.
- Go's RE2-based regular expressions avoid catastrophic backtracking.

### Important limitations and review points

1. **Fail-open missing-binary hook:** if the hook cannot find `aegis` or `./bin/aegis`, it prints a warning and exits `0`, allowing the commit.
2. **Active verification transmits credentials:** opt-in does not eliminate the sensitivity of sending tokens to third-party provider endpoints.
3. **AWS verification is heuristic:** the implementation uses a zero placeholder signature and infers key recognition from AWS error text; it does not prove that the corresponding secret-access key is valid.
4. **History scan path association is approximate:** `git rev-list --all --objects` maps object hashes to paths, while a blob can appear at multiple paths or across multiple commits.
5. **Ignore defaults may omit relevant files:** lockfiles and test files are broadly excluded by default, which reduces noise but can hide credentials in those artifacts.
6. **Single-line suppression is powerful:** any matching ignore phrase suppresses the complete line, so suppression governance matters.
7. **Rule/schema drift exists:** the platform specification describes statuses such as `NOT_VERIFIED`, `ACTIVE`, `UNVERIFIABLE`, and `ERROR`, while the Go model uses values such as `UNVERIFIED`, `ACTIVE_LEAK`, `REVOKED_DEAD`, and `VERIFICATION_ERROR`.
8. **Documentation and implementation drift:** README examples refer to a `lint` target that is not present in the Makefile, and some documented command/flag contracts differ from the actual entrypoint.
9. **Repository identity drift:** the repository is `Ilyankhan/aegis-cli`, but module paths, release links, installer repository names, and README links use `Ilyan321/aegis-cli`.

---

## 19. Documentation-to-Code Inconsistencies

This section is important for research use because it distinguishes intended architecture from implemented behavior.

### Go version mismatch

- `go.mod` and CI use Go `1.23`.
- The README badge says Go `1.27+`.
- These are not equivalent claims; the repository's build configuration currently targets Go 1.23.

### Default failure threshold mismatch

- `main.go` defines `--fail-on` default as `critical`.
- `NewDefaultScanOptions` also uses `CRITICAL`.
- The platform specification says the default is `LOW`.
- The README describes a policy intended to avoid false-alarm breaks but does not consistently specify the actual default.

### Verification status mismatch

- The platform specification names statuses like `NOT_VERIFIED`, `ACTIVE`, `UNVERIFIABLE`, `SKIPPED`, and `ERROR`.
- The Go model uses `UNVERIFIED`, `ACTIVE_LEAK`, `REVOKED_DEAD`, `VERIFICATION_ERROR`, and `SKIPPED`.
- Platform consumers must normalize these values or the contract must be revised.

### Architecture claims versus implementation

- The PRD references AST context awareness, but the implementation is line-oriented and regular-expression/heuristic based; no AST parser is present.
- The PRD references an Aho–Corasick automaton, but the implementation uses prefix strings and `strings.Contains`/`strings.Index`.
- The PRD describes direct Git pack/loose-object traversal, while the implementation invokes `git rev-list` and `git cat-file --batch`.
- The PRD discusses exponential backtracking in other regex engines; the implementation correctly uses Go's RE2-style engine, but there is no custom DFA engine.

### CI and Makefile mismatch

- README instructs users to run `make lint`.
- The Makefile has no `lint` target.
- `.golangci.yml` exists, but the GitHub Actions workflow does not invoke `golangci-lint`.

### Repository naming mismatch

The checked-in code and many documentation links refer to `github.com/Ilyan321/aegis-cli`, while the analyzed repository URL is `github.com/Ilyankhan/aegis-cli`. This can affect `go install`, release downloads, badges, and provenance.

---

## 20. Research-Relevant Engineering Skills Demonstrated

This repository demonstrates or attempts to demonstrate the following skills:

### Go systems programming

- package-oriented architecture;
- standard-library-only implementation;
- command-line process design;
- goroutines, channels, WaitGroups, and mutexes;
- streaming readers and bounded buffers;
- filesystem and permission management;
- cross-compilation and static linking.

### Secure software engineering

- secret masking;
- raw-secret exclusion from serialized output;
- deterministic fingerprints;
- provider-specific credential classification;
- explicit network opt-in;
- read-only verification endpoints;
- timeout and rate-limit controls;
- remediation and blast-radius modeling;
- safe Git hook replacement and restoration.

### Detection engineering

- regular-expression signature design;
- prefix pre-filtering;
- entropy calculation;
- alphabet classification;
- low-variance and sequence rejection;
- placeholder/test-context analysis;
- false-positive suppression;
- duplicate finding elimination.

### Git internals and DevSecOps

- unified diff parsing;
- staged-index analysis;
- merge-base range scanning;
- reachable-object enumeration;
- `git cat-file --batch` streaming;
- pre-commit hook integration;
- CI exit-code contracts;
- structured JSON ingestion.

### Platform integration

- authenticated CLI login;
- environment/file configuration precedence;
- HTTP API synchronization;
- repository/branch/commit metadata extraction;
- deterministic incident deduplication.

### Testing and quality

- unit tests;
- subprocess-based CLI tests;
- race detector usage;
- HTTP mock servers;
- benchmark tests;
- cross-package integration behavior;
- static-analysis configuration.

---

## 21. Recommended NotebookLM Research Questions

1. How does Aegis balance detection recall against false-positive reduction, and which heuristics could create blind spots?
2. Is the deterministic finding hash stable enough for security-incident deduplication when files move, lines change, or the same secret appears in multiple commits?
3. What are the privacy and governance implications of optional active verification, especially for AWS, OpenAI, GitHub, and other providers?
4. How should the platform normalize the discrepancy between the documented JSON contract and the actual Go verification-status values?
5. Does the pre-commit hook's fail-open behavior undermine the claimed security boundary when a binary is missing or installation paths differ?
6. How would a native Git object traversal compare with the current `git rev-list` plus `git cat-file` implementation in performance and correctness?
7. Which provider rules are most vulnerable to token-format changes, prefix collisions, or false positives?
8. What would be required to add SARIF output while preserving the no-raw-secret guarantee?
9. How should the project resolve the repository-name, Go-version, default-threshold, and `make lint` documentation inconsistencies?
10. Which claims in the PRD are implemented, partially implemented, or aspirational?

---

## 22. Concise End-to-End Mental Model

Aegis is a statically compiled Go CLI that accepts a scan target or Git mode, applies ignore rules, streams text through a binary/size gate, evaluates provider-specific regular expressions and entropy heuristics, creates masked deterministic findings, optionally verifies supported credentials against provider APIs, renders console or JSON output, optionally synchronizes the report to a cloud platform, and exits with a severity-aware status code.

Its strongest architectural qualities are low dependency count, clear package boundaries, deterministic reporting, Git-aware workflows, and strong test coverage around detection and verification. Its most important engineering risks are documentation drift, fail-open hook behavior, heuristic active verification, broad ignore defaults, and mismatch between the intended platform contract and the currently serialized Go model.

---

## 23. Source Files Used for This Context

- `README.md`
- `PRD.md`
- `AEGIS_CLI_SPEC.md`
- `go.mod`
- `Makefile`
- `.aegisignore`
- `.golangci.yml`
- `.github/workflows/ci.yml`
- `.github/workflows/release.yml`
- `cmd/aegis/main.go`
- `cmd/aegis/main_test.go`
- `internal/analyzer/engine.go`
- `internal/analyzer/entropy.go`
- `internal/analyzer/heuristics.go`
- `internal/analyzer/rules.go`
- `internal/analyzer/engine_test.go`
- `internal/analyzer/heuristics_test.go`
- `internal/config/ignore.go`
- `internal/config/settings.go`
- `internal/config/cloud.go`
- `internal/git/diff.go`
- `internal/git/history.go`
- `internal/git/hook.go`
- `internal/reporter/console.go`
- `internal/reporter/json.go`
- `internal/reporter/reporter_test.go`
- `internal/validator/verifier.go`
- provider verifier implementations under `internal/validator/`
- `internal/validator/verifier_test.go`
- `pkg/models/finding.go`
- `install.sh`
- `install.ps1`

