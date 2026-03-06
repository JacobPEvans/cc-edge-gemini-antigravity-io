# Gemini CLI & Antigravity Pack For Edge

## Overview

This Cribl Edge pack collects telemetry from 8 file monitor sources and 1 OTLP receiver across two Google AI developer tools and forwards it to a Cribl Stream worker group for indexing, analysis, and search:

### Gemini CLI

1. **Chat sessions** — `~/.gemini/tmp/{projectHash}/chats/session-*.json` — Full conversation transcripts with messages, tool calls, model thoughts, and per-message token usage
2. **User logs** — `~/.gemini/tmp/{projectHash}/logs.json` — User interaction message logs with session IDs and timestamps
3. **Settings** — `~/.gemini/settings.json` — Configuration snapshots including tool permissions, security settings, and context files
4. **Projects** — `~/.gemini/projects.json` — Project metadata and Google Cloud project associations

### Gemini CLI OpenTelemetry

5. **OTLP receiver** — `0.0.0.0:4317` (gRPC) — Native OpenTelemetry traces from Gemini CLI when `GEMINI_TELEMETRY_ENABLED=true` and `GEMINI_TELEMETRY_OTLP_ENDPOINT=http://localhost:4317`

### Antigravity IDE

6. **Application logs** — `~/Library/Application Support/Antigravity/logs/**/*.log` — Main process, extension host, language server, auth, telemetry, CloudCode, Chrome DevTools MCP, renderer performance, and crash logs
7. **Agent brain** — `~/.gemini/antigravity/brain/**/*.md` — Agent task files, implementation plans, and walkthroughs with resolved version history
8. **Annotations** — `~/.gemini/antigravity/annotations/*.pbtxt` — Annotation protobuf text metadata
9. **Code tracker** — `~/.gemini/antigravity/code_tracker/**/*` — Code tracking snapshots and file state across active and historical sessions

## Architecture

```
Gemini CLI ──writes──> ~/.gemini/tmp/{hash}/chats/session-*.json
    |                                   |
    |                        Cribl Edge (file monitor)
    |                                   |
    └──OTLP gRPC──> Cribl Edge :4317 (OTLP receiver)
                                        |
Antigravity IDE ──writes──> ~/Library/Application Support/Antigravity/logs/**/*.log
       |                                |
       └──writes──> ~/.gemini/antigravity/brain/**/*.md
                                        |
                             Cribl Edge (file monitor)
                                        |
                             Cribl HTTP --> Cribl Stream Worker Group
```

## Data Sources

### Gemini CLI Chat Sessions

The richest data source. Each session JSON file contains the full conversation including:

- **User messages** — The exact prompts sent to the model
- **Gemini responses** — Model output with content blocks
- **Tool calls** — File reads, writes, shell commands with full arguments and results
- **Thoughts** — Model reasoning steps with subjects and descriptions
- **Token usage** — Per-message breakdown: input, output, cached, thoughts, tool, total
- **Model identification** — Which Gemini model was used (e.g., `gemini-3-pro-preview`)
- **Session metadata** — Session ID, project hash, timestamps

Use this data for:

- **Audit and compliance** — Complete record of every AI interaction
- **Cost tracking** — Token consumption per session, project, and model
- **Tool usage analysis** — What tools the agent invokes and how often
- **Productivity metrics** — Session duration, message counts, tool call patterns

### Gemini CLI User Logs

Lighter-weight logs recording only user-initiated messages:

- **Session ID** — Links messages to their full session
- **Message ID** — Sequential message number within a session
- **Type** — Always `user` in this log
- **Message** — The raw prompt text
- **Timestamp** — ISO 8601

### Gemini CLI OpenTelemetry

Gemini CLI has native OpenTelemetry support. When enabled, it exports traces via OTLP gRPC to a configurable endpoint. This pack includes an OTLP receiver on port 4317 to collect these traces directly.

**Enable via environment variables:**

```bash
export GEMINI_TELEMETRY_ENABLED=true
export GEMINI_TELEMETRY_OTLP_ENDPOINT=http://localhost:4317
```

**Or via `~/.gemini/settings.json`:**

```json
{
  "telemetry": {
    "enabled": true,
    "otlpEndpoint": "http://localhost:4317",
    "otlpProtocol": "grpc",
    "logPrompts": true
  }
}
```

Use this data for:

- **Distributed tracing** — End-to-end trace visibility across Gemini CLI tool invocations
- **Latency analysis** — Model response times, tool execution duration
- **Error tracking** — Failed operations with full trace context
- **Correlation** — Link OTLP traces to file-based session data via session IDs

### Antigravity IDE Application Logs

Antigravity (Google's AI IDE) writes timestamped session log directories containing:

```
Log File                                    Description
---------------------------------------------------------------------
main.log                                    App lifecycle, perf profiling sessions
auth.log                                    OAuth token changes and auth events
telemetry.log                               Internal telemetry data
cloudcode.log                               Cloud Code API endpoint configuration
rendererPerf.log                            Renderer performance baseline and long tasks
sharedprocess.log                           Shared process events
terminal.log                                Terminal session events
artifacts.log                               Artifact generation events
ptyhost.log                                 PTY host process lifecycle
remoteTunnelService.log                     Remote tunnel configuration
window1/exthost/exthost.log                 Extension host startup and activation
window1/exthost/google.antigravity/*.log    Language server (Go), crash logs
window1/exthost/google.chrome-devtools-mcp/*.log  Chrome DevTools MCP server
window1/exthost/vscode.git/Git.log          Git operations with timing
window1/exthost/vscode.github/*.log         GitHub integration
window1/renderer.log                        Renderer warnings and errors
window1/network.log                         Network activity
```

Log format is: `YYYY-MM-DD HH:MM:SS.mmm [level] message`

### Antigravity Agent Brain

The Antigravity agent maintains a "brain" directory with per-session UUID folders containing:

- **task.md** — Current task description with resolved versions tracking iterative refinement
- **implementation_plan.md** — Step-by-step implementation plans with metadata
- **walkthrough.md** — Code walkthroughs and explanations
- **\*.metadata.json** — Metadata for each document including revision counts

Each document has `.resolved` and `.resolved.N` variants showing the evolution of the agent's thinking.

---

## Requirements

- **Cribl Edge** 4.13.0+
- **Gemini CLI** installed for the local user (`/opt/homebrew/bin/gemini` or equivalent)
- **Antigravity IDE** installed (`/Applications/Antigravity.app` on macOS)
- **Supported platforms:** macOS (Sonoma 14+, Sequoia 15+, Tahoe 26)
- **Filesystem permissions:** The Cribl Edge process must have read access to `~/.gemini/` and `~/Library/Application Support/Antigravity/` (if using the respective inputs)

---

## Setup

### Step 1: Set the `GEMINI_HOME` Environment Variable

All file monitor paths use `$GEMINI_HOME/.gemini/<subdir>` or `$GEMINI_HOME/Library/Application Support/Antigravity/<subdir>` to resolve their target directories. Set `GEMINI_HOME` to the **home directory** of the user that runs Gemini CLI and/or Antigravity.

```bash
# macOS
export GEMINI_HOME=/Users/<user>
```

> After setting the variable, restart the Cribl Edge service for it to take effect.

### Step 2: Grant Filesystem Permissions

On macOS, Cribl Edge is typically installed under the current user account. If Edge is running as the same user that owns the Gemini/Antigravity files, no additional permission changes are needed.

If Edge runs as a different user, grant read access using macOS ACLs:

```bash
# Grant read access to Gemini CLI data
chmod +a "cribl allow read,readattr,readextattr,readsecurity,list,search" /Users/<user>/.gemini
chmod -R +a "cribl allow read,readattr,readextattr,readsecurity,list,search" /Users/<user>/.gemini/

# Grant read access to Antigravity logs
chmod +a "cribl allow read,readattr,readextattr,readsecurity,list,search" "/Users/<user>/Library/Application Support/Antigravity"
chmod -R +a "cribl allow read,readattr,readextattr,readsecurity,list,search" "/Users/<user>/Library/Application Support/Antigravity/logs/"
```

**Verification:**

```bash
# Verify Gemini CLI sessions are readable
ls /Users/<user>/.gemini/tmp/*/chats/

# Verify Antigravity logs are readable
ls "/Users/<user>/Library/Application Support/Antigravity/logs/"
```

---

## Pack Components

These are preconfigured in the pack. This section is for reference — no action is needed here unless you want to customize the defaults.

### File Monitor Inputs

All file monitors resolve paths via `$GEMINI_HOME`. Each sets a `datatype` metadata field for downstream routing.

| Input | Path | Filter | Recursive | Interval |
|---|---|---|---|---|
| `gemini-cli-sessions` | `.gemini/tmp` | `session-*.json` | Yes | 10s |
| `gemini-cli-logs` | `.gemini/tmp` | `logs.json` | Yes | 30s |
| `gemini-cli-settings` | `.gemini` | `settings.json` | No | 120s |
| `gemini-cli-projects` | `.gemini` | `projects.json` | No | 120s |
| `antigravity-app-logs` | `Library/Application Support/Antigravity/logs` | `*.log` | Yes | 10s |
| `antigravity-brain` | `.gemini/antigravity/brain` | `*.md` | Yes | 60s |
| `antigravity-annotations` | `.gemini/antigravity/annotations` | `*.pbtxt` | No | 60s |
| `antigravity-code-tracker` | `.gemini/antigravity/code_tracker` | `*.md, *.sh, *.nix, *.yaml, *.json` | Yes | 60s |

### OTLP Input

| Input | Type | Host | Port | Protocol | TLS |
|---|---|---|---|---|---|
| `gemini-cli-otel` | OpenTelemetry | `0.0.0.0` | `4317` | gRPC | Disabled |

Receives native OpenTelemetry traces from Gemini CLI. Enable with `GEMINI_TELEMETRY_ENABLED=true` and `GEMINI_TELEMETRY_OTLP_ENDPOINT=http://localhost:4317`.

### Output: `default`

- **Type:** Cribl HTTP
- **Compression:** gzip
- **Concurrency:** 5

Forwards events to the Cribl Stream worker group for further routing and delivery to your destination of choice (Splunk, S3, Elastic, etc.).

---

## Reference: Gemini CLI Session Schema

Each `session-*.json` file is a standalone JSON document with this structure:

```
Field                        Description
---------------------------------------------------------------------
sessionId                    Unique session identifier (UUID)
projectHash                  SHA-256 hash of the project directory
startTime                    ISO 8601 session start timestamp
lastUpdated                  ISO 8601 last activity timestamp
messages[]                   Array of conversation turns
  .id                        Unique message ID (UUID)
  .timestamp                 ISO 8601 message timestamp
  .type                      "user" or "gemini"
  .content                   Message text content
  .toolCalls[]               Tool invocations (gemini turns only)
    .id                      Tool call ID
    .name                    Tool name (read_file, write_file, shell, etc.)
    .args                    Tool arguments object
    .result[]                Tool execution results
    .status                  "success" or "error"
    .displayName             Human-readable tool name
    .description             Tool description
  .thoughts[]                Model reasoning steps (gemini turns only)
    .subject                 Thought topic
    .description             Reasoning detail
    .timestamp               When the thought occurred
  .model                     Model identifier (e.g., "gemini-3-pro-preview")
  .tokens                    Token usage breakdown (gemini turns only)
    .input                   Input tokens consumed
    .output                  Output tokens generated
    .cached                  Tokens served from cache
    .thoughts                Tokens used for reasoning
    .tool                    Tokens used for tool interactions
    .total                   Total tokens for this turn
```

## Reference: Antigravity Log Format

All Antigravity logs follow the VS Code log format:

```
YYYY-MM-DD HH:MM:SS.mmm [level] message
```

Levels: `info`, `warning`, `error`

### Key Log Sources

**Language Server (Go)**
- Process startup with PID and GOMAXPROCS
- HTTPS/HTTP port bindings on localhost
- Server initialization timing
- MCP URL discovery

**Extension Host**
- Extension activation order and timing
- Activation events and root causes
- Extension errors and missing modules

**Chrome DevTools MCP**
- MCP server URL and port
- Browser connection configuration

**Performance**
- Renderer baseline measurements
- Long task detection (>100ms threshold)
- Profiling session start/stop

---

## Troubleshooting

### No events flowing

1. Verify `$GEMINI_HOME` is set correctly and points to the user's home directory.
2. Check that the file monitor discovered files in the Cribl Edge logs.
3. Verify the route is not disabled in the Cribl UI.
4. Confirm the output destination is reachable.

### No Gemini CLI session files found

1. Verify Gemini CLI has been used at least once: `ls ~/.gemini/tmp/*/chats/`
2. Check that session files exist: `find ~/.gemini/tmp -name "session-*.json" | head -5`
3. Ensure `$GEMINI_HOME` resolves correctly in the Cribl Edge environment.

### No Antigravity logs found

1. Verify Antigravity has been launched at least once: `ls ~/Library/Application\ Support/Antigravity/logs/`
2. Check that log directories contain `.log` files: `find ~/Library/Application\ Support/Antigravity/logs/ -name "*.log" -size +0c | head -10`
3. Note that some Antigravity log files may be empty (0 bytes) if the feature wasn't used in that session.

### No OTLP data from Gemini CLI

1. Verify telemetry is enabled: `echo $GEMINI_TELEMETRY_ENABLED` should return `true`.
2. Verify the endpoint: `echo $GEMINI_TELEMETRY_OTLP_ENDPOINT` should return `http://localhost:4317`.
3. Alternatively check `~/.gemini/settings.json` for a `telemetry` block with `enabled: true`.
4. Confirm port 4317 is not in use by another collector: `lsof -i :4317`.
5. Check Cribl Edge logs for OTLP receiver bind errors.

### Stale file tracking

Cribl Edge tracks file state in its kvstore. If you need to re-ingest files from the beginning, stop the worker, clear the relevant kvstore directories, and restart.

- macOS: `/opt/cribl/state/kvstore/default/file_gemini-cli-*/` and `/opt/cribl/state/kvstore/default/file_antigravity-*/`

---

## Release Notes

- **1.1.0** — 2026-03-06
  - Added OTLP gRPC receiver (port 4317) for Gemini CLI native OpenTelemetry traces
  - Added route for `gemini-cli-otel` input
  - Added telemetry configuration documentation (env vars and settings.json)
  - Added OTLP troubleshooting section
- **1.0.0** — 2026-03-06
  - Initial release
  - 4 Gemini CLI file monitor inputs (sessions, logs, settings, projects)
  - 4 Antigravity IDE file monitor inputs (app logs, brain, annotations, code tracker)
  - Per-input `datatype` metadata for downstream sourcetype mapping
  - `GEMINI_HOME` environment variable for path resolution
  - macOS ACL-based permission model
  - Sanitized sample data for all major event types
  - Automated `.crbl` pack release via GitHub Actions

## Authors

* Andrew Hendrix - <Andrewh@VisiCoreTech.com>
* Jacob Evans - <jevans@VisiCoreTech.com>

To contact us, please email <CriblPacks@VisiCoreTech.com>.

## Contributing to the Pack

To contribute to this Pack, or to report any issues or enhancement requests, please connect with **VisiCore Tech** on [Cribl Community Slack](https://cribl-community.slack.com) or email us at: <CriblPacks@visicoretech.com>.

## License
---
This Pack uses the following license: [`Apache 2.0`](https://github.com/criblio/appscope/blob/master/LICENSE)
