# ABAP Accelerator CLI

`abapctl` is a standalone binary that automates SAP development workflows. Think of it as [ABAP Accelerator](https://github.com/aws-solutions-library-samples/guidance-for-deploying-sap-abap-accelerator-for-amazon-q-developer) in batch (headless) mode. Optimized for AI agents and usable by humans from the terminal. Works with any ADT-enabled SAP system (ECC, S/4HANA).

## 1. Architecture

*Every command runs as a direct call to your SAP system, with each call captured both in your SAP audit trail and in a local log under `.abapctl/logs/`.*

```
abapctl ──HTTPS──►  /sap/bc/adt/...  ──►  SAP system
   │                   ADT REST
   └─ .abapctl.json (connection, defaults)
```

## 2. Getting started

*Install the binary and point it at a development SAP system.*

### Get the binary

Download the binary for your platform and put it on your `PATH`. Each binary is self-contained.

<details>
<summary><b>Linux (x86_64)</b></summary>

```bash
curl -fsSLo abapctl https://github.com/aws-for-sap/Automate-SAP-development-workflows-using-ABAP-CTL/releases/latest/download/abapctl-linux-x64
chmod +x abapctl
sudo mv abapctl /usr/local/bin/
abapctl --version
```
</details>

<details>
<summary><b>macOS (Apple Silicon)</b></summary>

```bash
curl -fsSLo abapctl https://github.com/aws-for-sap/Automate-SAP-development-workflows-using-ABAP-CTL/releases/latest/download/abapctl-macos-arm64
chmod +x abapctl
xattr -d com.apple.quarantine abapctl 2>/dev/null || true
sudo mv abapctl /usr/local/bin/
abapctl --version
```
</details>

<details>
<summary><b>macOS (Intel)</b></summary>

```bash
curl -fsSLo abapctl https://github.com/aws-for-sap/Automate-SAP-development-workflows-using-ABAP-CTL/releases/latest/download/abapctl-macos-x64
chmod +x abapctl
xattr -d com.apple.quarantine abapctl 2>/dev/null || true
sudo mv abapctl /usr/local/bin/
abapctl --version
```
</details>

<details>
<summary><b>Windows (x64)</b></summary>

```powershell
Invoke-WebRequest -Uri https://github.com/aws-for-sap/Automate-SAP-development-workflows-using-ABAP-CTL/releases/latest/download/abapctl-windows-x64.exe -OutFile abapctl.exe
.\abapctl.exe --version
```

Move `abapctl.exe` somewhere on your `PATH`.
</details>

<details>
<summary><b>Tab completion (one-time)</b></summary>

```bash
# bash
echo 'source <(abapctl completion bash)' >> ~/.bashrc

# zsh
echo 'source <(abapctl completion zsh)' >> ~/.zshrc

# fish
abapctl completion fish > ~/.config/fish/completions/abapctl.fish
```

Reopen your terminal. Tab completes commands, subcommands, and flag values.
</details>

### Configure

Create a config file in your project root:

```bash
abapctl init
```

This writes a `.abapctl.json` template. Edit it to point at your development or sandbox system:

```jsonc
{
  "connections": {
    "dev": {
      "host": "sap.example.com",
      "port": 443,
      "sid": "S4H",
      "client": "100",
      "secure": true,
      "username": "DEVELOPER",
      "password_env": "SAP_PASSWORD",
      "language": "EN"
    }
  },
  "defaults": { "connection": "dev" }
}
```

`port` is optional. It defaults to `443` when `secure: true` and `8000` when `secure: false`. Set it explicitly if your SAP system listens on a non-standard ADT port (common for on-prem instances on the 8000-8099 or 50000-range).

The password lives in an environment variable. The config file only stores the variable's name, never the password itself. Source the password from your secret manager, for example:

```bash
export SAP_PASSWORD=$(aws secretsmanager get-secret-value --secret-id sap/dev --query SecretString --output text)
```

Verify the connection:

```bash
export SAP_PASSWORD='...'
abapctl system-check
abapctl object info ZCL_FOO       # first read confirms it works
```

Most commands accept `-c <connection-name>` to override the default. Set up multiple connections in `connections` and switch with `abapctl config set-connection <name>`.

## 3. Read source

*Pull source code, metadata, and history for any object.*

```bash
abapctl source get ZCL_FOO                          # main include
abapctl source get ZCL_FOO --include testclasses    # specific class include
abapctl object info ZCL_FOO                         # metadata, includes, links
```

`source get` writes to stdout. Pipe it where you need it. For an entire package, see [Workspaces](#8-workspace-like-git-clone-for-an-sap-package).

## 4. Search and navigate

*Find objects by name, browse a package, jump to definitions, and use compiler-authoritative completion.*

```bash
abapctl object search 'ZCL_FOO*' --type CLAS
abapctl object tree ZFINANCE
abapctl code definition ZCL_FOO --line 42 --col 18
abapctl code references ZCL_FOO --line 42 --col 18
abapctl code complete ZCL_FOO --line 47 --col 47    # same backend as Eclipse Ctrl+Space
```

`code complete` calls SAP's `abapsource/codecompletion/proposal` endpoint, the same call Eclipse uses for its Ctrl+Space dropdown. The response is the SAP compiler's view of valid identifiers at that cursor position.

## 5. Edit source

*Modify objects with the full lock / write / unlock / activate cycle handled for you.*

`abapctl source put` handles the full lock / write / unlock / activate cycle. If activation fails, the lock is released in a `finally` block so the object isn't left locked.

```bash
abapctl source get ZCL_FOO > zcl_foo.clas.abap
# edit zcl_foo.clas.abap in your editor
abapctl source put ZCL_FOO --file zcl_foo.clas.abap --dry-run    # preview, no SAP write
abapctl source put ZCL_FOO --file zcl_foo.clas.abap              # commit
```

For multiple files in one go:

```bash
abapctl source push zcl_foo.clas.abap zif_bar.intf.abap zprog_baz.prog.abap
```

`source push` locks each object, writes, unlocks, then activates everything in one batch. Useful for RAP stacks where activation order matters.

To skip activation and leave the object inactive, pass `--no-activate`. To pretty-print before saving:

```bash
cat zcl_foo.clas.abap | abapctl source format > zcl_foo.formatted.abap
```

## 6. Quality checks

*Run syntax, ATC, ABAP Unit, and CDS-DDL checks against the active version or a draft buffer.*

```bash
abapctl check syntax ZCL_FOO
abapctl check atc ZCL_FOO --variant ABAP_CLOUD_READINESS
abapctl check unit ZCL_FOO
abapctl check cds-syntax I_MyView
```

`abapctl check atc-variants` lists the variants available on your system. To set a default variant for `clean-core` workflows, edit `clean_core.atc_variant` in `.abapctl.json`.

For per-finding documentation URLs and fix-availability flags (manual / automatic / pseudo / AI-fixable), use:

```bash
abapctl check atc-quickfix ZCL_FOO --variant ABAP_CLOUD_READINESS --json
```

This surfaces the metadata SAP already ships in every ATC worklist that the regular `check atc` projection drops.

## 7. Manage transports

*Create, inspect, release, and delete transport requests.*

```bash
abapctl transport list                          # your open requests
abapctl transport info ZCL_FOO                  # which TR holds this object
abapctl transport create --description '...'
abapctl transport get K9A123                    # full TR detail with tasks/objects
abapctl transport release K9A123 --dry-run      # preview
abapctl transport release K9A123
abapctl transport delete K9A123
```

`transport release`, `transport delete`, and `transport create` accept `--dry-run` to validate inputs without making the SAP call.

## 8. Workspace: like `git clone` for an SAP package

*Sync an entire SAP package to a local directory and work against it the way you would a git repo.*

For sustained work on a package, sync it to a local directory the way you'd clone a git repo:

```bash
abapctl workspace init ZFINANCE
```

This discovers all objects in `ZFINANCE`, downloads each to `abap/S4H/ZFINANCE/<object>.<type>.abap`, and writes a manifest tracking baseline checksums.

Filenames follow the [ABAP File Formats](https://github.com/SAP/abap-file-formats) convention used by abapGit and SAP's own tooling, so workspace contents are interoperable with the broader ABAP ecosystem.

Day-to-day commands:

```bash
abapctl workspace status S4H/ZFINANCE      # what is modified, missing, in conflict
abapctl workspace diff S4H/ZFINANCE        # unified diff against SAP baseline
abapctl workspace push S4H/ZFINANCE        # upload local changes
abapctl workspace pull S4H/ZFINANCE        # download SAP-side changes (with conflict detection)
abapctl workspace refresh S4H/ZFINANCE     # discover objects added since init
```

The directory structure is SID-aware. A workspace initialized against `S4H` refuses to push against `S4P`. This prevents the classic "I uploaded my dev changes to QA" mistake when you're juggling connections.

To stop tracking specific objects:

```bash
abapctl workspace remove S4H/ZFINANCE ZCL_OLD ZCL_DEPRECATED --delete --yes
```

## 9. Working with AI agents

*Pair abapctl with [Kiro CLI](https://kiro.dev/docs/cli/headless/) for AI-driven SAP development.*

The integration contract is `abapctl tools list --json`. It returns the full command catalog with parameter schemas, descriptions, and safety flags (read-only / destructive / idempotent). Drop that JSON into your agent's tool registry and it can plan, route, and reason about every abapctl command.

For agent calls, four flags do most of the work:

| Flag | Purpose |
|---|---|
| `--json` | Stable, machine-readable output on stdout (progress prose stays on stderr). Errors carry codes like `SAP_AUTH_ERROR`, `SAP_HTTP_ERROR`, `LOCK_HELD_BY_USER`. |
| `--dry-run` | Preview destructive commands without writing to SAP. The agent presents the preview; the human approves; a second call without `--dry-run` commits. |
| `--yes` | Skip the interactive confirmation prompt on destructive commands. Use after a human has approved the dry-run preview. |
| `--session-file <path>` | Reuse one SAP login across many calls. Saves ~8s per command on slow-login systems. Stale sessions auto-recover. |

A typical agent loop looks like this:

```bash
SESSION=$(mktemp)

# Read freely.
abapctl object info ZCL_FOO --json --session-file $SESSION
abapctl check atc ZCL_FOO --json --session-file $SESSION | jq '.findings[]'

# Propose a write, show the human, commit.
abapctl source put ZCL_FOO --file new.clas.abap --dry-run --json --session-file $SESSION
# ...human reviews the proposed change...
abapctl source put ZCL_FOO --file new.clas.abap --yes --session-file $SESSION
```

`--dry-run` works on `source put`, `source push`, all `workspace` writes, all `create` commands, transport release/delete, object delete/activate, service publish/unpublish, `clean-core apply`, and `bdef create`. Browse the full catalog with safety flags in [Commands](#11-commands).

This repo ships a ready-made Kiro skill at [`.kiro/skills/sap-abap/`](./.kiro/skills/sap-abap/). Clone or copy this into your project and Kiro auto-discovers the skill on the next session.

## 10. Assess and remediate for Clean Core

*Classify a package against Clean Core levels and drive the fix loop with an AI agent.*

SAP Clean Core is about decoupling custom extensions from SAP standard code so that S/4HANA upgrades stay safe. Custom code that depends on internal or non-released SAP APIs creates upgrade risk; the [Clean Core Extensibility whitepaper](https://www.sap.com/documents/2024/09/20aece06-d87e-0010-bca6-c68f7e60039b.html) defines four compliance levels that quantify it.

abapctl runs the assessment, prepares per-object fix context, and applies the fixes. The fix step itself is where an AI agent does the work, using abapctl to read, check, and validate against the live SAP system. A sample remediation knowledge base for the fix loop lives in the [`clean-core/cc-kb/`](https://github.com/aws-for-sap/agentic-ai-guidance-for-SAP-use-cases/tree/main/clean-core/cc-kb) directory of the [agentic-ai-guidance-for-SAP-use-cases](https://github.com/aws-for-sap/agentic-ai-guidance-for-SAP-use-cases) repo.

### 1. Assess

```bash
abapctl clean-core assess ZFINANCE -c s4h
```

Discovers every object in the package, runs ATC with the cloud-readiness variant, and classifies each object:

| Level | ATC result | What it means | Action |
|---|---|---|---|
| **A** | No findings | Fully compliant with Clean Core | Cloud-ready, no action needed |
| **B** | Info only | Uses documented extension points | Acceptable, low upgrade risk |
| **C** | Warnings | Uses internal or undocumented APIs | Conditionally clean. Verify before upgrades. |
| **D** | Errors | Non-released APIs, modifications, or blocked patterns | Requires remediation |

Output goes to `clean-core/{SID}/{PACKAGE}/`:

- `reports/SUMMARY.md`: per-object table with level and finding count
- `reports/<OBJECT>_atc.md`: per-object Markdown report
- `state/discovery.json`: full object inventory
- `state/progress.json`: resumable checkpoint
- `state/<OBJECT>_findings.json`: raw structured findings

The run is idempotent and resumable. Re-running picks up where it left off; pass `--force` to start fresh. Exit code `0` means clean (A/B), `1` means findings exist (C/D), `2` means error.

For a cross-package roll-up:

```bash
abapctl clean-core executive
```

### 2. Prep

```bash
abapctl clean-core prep S4H/ZFINANCE
```

For each D and C object, downloads the source plus per-finding fix context. Output lands under `clean-core/{SID}/{PACKAGE}/fix-context/`:

- `<obj>.<type>.abap`: editable source (abapGit/AFF naming)
- `.context/<obj>.findings.json`: structured findings (line, message, deprecated API, suggested successor, doc URL)
- `.context/<obj>.context.md`: human-readable fix brief

### 3. Fix (AI agent loop)

*Example flow. Adapt to your agent's capabilities.*

This is where Kiro CLI does the work. For each priority-1 finding on a Level D object, the agent searches for a released-API replacement using three sources, stopping at the first hit:

1. **ATC tags on the finding.** Every ATC finding ships per-finding metadata (`refObjectName`, `refObjectType`, `Successors:` info). When the SAP system has already classified the deprecated API and identified its successor, that data is right there in the finding.
2. **SAP Cloudification Library reference JSONs.** Run `abapctl reference update` once, and `objectReleaseInfoLatest.json` + `objectClassifications_SAP.json` land under `.abapctl/reference/sap-api-reference/`. These cover the broad catalog of deprecated APIs and their cloud-ready successors.
3. **Live CDS view query.** When the first two are inconclusive, `abapctl release-state lookup <api>` queries `I_APISWITHCLOUDDEVSUCCESSOR` on the connected SAP system directly. Always current to whatever release the target system is on.

When the agent reports the object as clean, the human reviews the diff, then runs apply.

### 4. Apply

```bash
abapctl clean-core apply S4H/ZFINANCE --dry-run    # preview every write
abapctl clean-core apply S4H/ZFINANCE              # commit
```

Locks each object, uploads the fixed source, unlocks, and activates. Failures roll back via the lock-finally guard, so a partial run never leaves SAP in a half-modified state. Re-run to continue from the failure point.

## 11. Commands

*Full command catalog with safety annotations. Run `abapctl tools list` to regenerate this list against your installed binary.*

`READ` = the command only reads from SAP. `DESTR` = the command can mutate SAP. `IDEMP` = re-running produces the same result. `PARAMS` is the count of options the command accepts.

<details>
<summary><b>Show all 113 commands</b></summary>

```
COMMAND                          READ   DESTR  IDEMP  PARAMS
──────────────────────────────── ───── ───── ───── ──────
object info                      yes    —      yes    2
object tree                      yes    —      yes    2
object types                     yes    —      yes    1
object inactive                  yes    —      yes    1
object delete                    —      YES    —      5
object history                   yes    —      yes    3
object search                    yes    —      yes    5
object activate                  —      —      yes    3
object path                      yes    —      yes    2
clean-core assess                yes    —      yes    4
clean-core report                —      —      yes    1
clean-core executive             —      —      yes    1
clean-core prep                  —      —      yes    2
clean-core apply                 —      YES    —      5
check syntax                     yes    —      yes    3
check atc                        yes    —      yes    3
check atc-quickfix               yes    —      yes    5
check unit                       yes    —      yes    3
check atc-variants               yes    —      yes    1
check cds-syntax                 yes    —      yes    3
check atc-exempt-proposal        yes    —      yes    2
check atc-exempt                 —      —      —      5
package list                     yes    —      yes    2
package exists                   yes    —      yes    2
package lookup                   yes    —      yes    3
reference update                 —      —      yes    0
transport info                   yes    —      yes    3
transport create                 —      —      —      5
transport list                   yes    —      yes    2
transport get                    yes    —      yes    2
transport release                —      YES    —      6
transport delete                 —      YES    —      4
create class                     —      —      —      8
create function-group            —      —      —      8
create annotation-definition     —      —      —      6
create table                     —      —      —      8
create structure                 —      —      —      8
create data-element              —      —      —      20
create domain                    —      —      —      15
create program                   —      —      —      8
create interface                 —      —      —      8
create include                   —      —      —      8
create ddl-source                —      —      —      8
create service-definition        —      —      —      8
create access-control            —      —      —      8
create metadata-extension        —      —      —      8
create function-module           —      —      —      9
create function-group-include    —      —      —      9
create message-class             —      —      —      8
create auth-field                —      —      —      8
create auth-object               —      —      —      8
create service-binding           —      —      —      7
create package                   —      —      —      10
code definition                  yes    —      yes    7
code element-info                yes    —      yes    5
code references                  yes    —      yes    5
code snippets                    yes    —      yes    5
code complete                    yes    —      yes    9
refactor quickfix                —      YES    —      6
refactor rename                  —      YES    —      8
refactor extract-method          —      YES    —      9
refactor move                    —      YES    —      4
source get                       yes    —      yes    4
source put                       —      YES    —      9
source push                      —      YES    —      7
source format                    yes    —      yes    1
source format-settings           —      —      yes    3
data query                       yes    —      yes    4
data cds                         yes    —      yes    3
data sql                         yes    —      yes    4
cds annotations                  yes    —      yes    1
cds element-info                 yes    —      yes    5
cds repository-access            yes    —      yes    2
cds test-dependencies            yes    —      yes    4
service publish                  —      —      yes    3
service unpublish                —      YES    yes    4
ddic table-settings              yes    —      yes    2
ddic data-element                yes    —      yes    2
ddic domain                      yes    —      yes    2
ddic table-type                  yes    —      yes    2
ddic lock-object                 yes    —      yes    2
ddic type-group                  yes    —      yes    2
ddic structure                   yes    —      yes    2
ddic table                       yes    —      yes    2
ddic view                        yes    —      yes    2
bdef get                         yes    —      yes    2
bdef create                      —      —      —      8
enho on                          yes    —      yes    5
enho list                        yes    —      yes    5
config show                      yes    —      yes    0
config set-connection            —      —      yes    1
config add-connection            —      —      —      1
workspace init                   —      —      —      12
workspace status                 yes    —      yes    2
workspace list                   yes    —      yes    2
workspace pull                   —      —      yes    5
workspace push                   —      YES    —      8
workspace diff                   yes    —      yes    5
workspace add                    —      —      yes    5
workspace remove                 —      —      yes    6
workspace reset                  —      YES    yes    6
workspace refresh                —      —      —      9
release-state lookup             yes    —      yes    2
text-elements get                yes    —      yes    4
text-elements put                —      YES    —      8
init                             —      —      yes    2
system-check                     yes    —      yes    5
run                              —      —      —      2
tools list                       yes    —      yes    0
tools coverage                   yes    —      yes    3
discover                         yes    —      yes    2
recipes                          yes    —      yes    2
completion                       —      —      —      1
```

For machine-readable output with parameter schemas, run `abapctl tools list --json`.
</details>

## 12. References

abapctl's Clean Core remediation patterns are synthesized from SAP's published guidance:

- [SAP Clean Core Extensibility whitepaper](https://www.sap.com/documents/2024/09/20aece06-d87e-0010-bca6-c68f7e60039b.html): the four compliance levels and the cloud-readiness model
- [SAP-samples/abap-cheat-sheets](https://github.com/SAP-samples/abap-cheat-sheets): ABAP syntax patterns and modern-ABAP examples (Apache 2.0)
- [SAP/abap-atc-cr-cv-s4hc](https://github.com/SAP/abap-atc-cr-cv-s4hc): the SAP cloudification library (deprecated APIs and successors), fetched by `abapctl reference update` (Apache 2.0)
- [SAP/styleguides](https://github.com/SAP/styleguides): Clean ABAP style guidance (Apache 2.0)

## 13. License

abapctl is licensed under [MIT-0](./LICENSE) (MIT No Attribution).

The shipped binaries bundle the Node.js runtime and several MIT-licensed npm dependencies. See [THIRD-PARTY-NOTICES](./THIRD-PARTY-NOTICES.md) for full attributions.
