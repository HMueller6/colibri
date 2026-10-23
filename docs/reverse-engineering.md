# Reverse engineering with REA

[REA](https://rea.tools/) (Reverse Engineer Anything,
[morluto/rea](https://github.com/morluto/rea), MIT) is an MCP server and CLI
that gives a coding agent reverse-engineering tools: native binaries through
Ghidra, Hopper or IDA, JavaScript and Electron apps, .NET assemblies, Android
APKs, WebAssembly, websites and HAR or pcap captures, firmware, and model files
such as GGUF and safetensors. The analysis runs on this machine, and each result
carries the evidence and the limits behind it.

The agent's model is another matter: the agent sends REA's results to whatever
model it uses. With colibri serving that model, neither the target nor the
conversation leaves the machine ([below](#with-a-local-model)).

Get REA only from its official sources: the repository above,
[rea.tools](https://rea.tools/), and the
[`rea-agents`](https://www.npmjs.com/package/rea-agents) package on npm.

## Add it to your agent

The repository does not register REA for you: an MCP server is code the agent
runs, so each person adds it themselves. For Claude Code, run this in the
colibri folder:

```bash
claude mcp add --scope local rea -- npx -y rea-agents@6.7.0 mcp
```

Local scope keeps the entry in `~/.claude.json` under this folder's path, so
other projects do not load it and nothing lands in the repository (REA's
documentation advises against `--scope project`, which writes a `.mcp.json`
that gets committed). `claude mcp list` shows it, and the first start downloads
the package through `npx`. `claude mcp remove --scope local rea`, from the same
folder, takes it out again.

To have REA in every project instead, `npx rea-agents setup` registers it at
the user level for each supported agent it finds (Claude Code, Codex, Cursor,
Gemini CLI and others), adds REA's workflow instructions, and backs up every
configuration file it changes. Restart the agent afterwards.

What it needs:

- Node.js 22.19 or newer in the 22 line, 24.11 or newer in the 24 line, or 26
  and later, with npm.
- For deep native analysis, a Ghidra, Hopper or IDA you already have. REA does
  not ship them and grants no license for them. APKs need a JDK and JADX.
  Static JavaScript and Electron analysis needs nothing else.
- On Linux, a procps `ps` on the `PATH` for Ghidra, JADX, firmware and static
  JavaScript analysis.

REA's `binary_session` tool reports which of its tools this session can use.

**Windows** (native, not WSL): Claude Code cannot start `npx` directly there,
so the entry goes through `cmd`:

```powershell
claude mcp add --scope local rea -- cmd.exe /d /c npx -y rea-agents@6.7.0 mcp
```

**Without an agent**, the CLI runs one analysis at a time; `--help` lists the
commands:

```bash
npx -y rea-agents@6.7.0 --help
npx -y rea-agents@6.7.0 analyze-javascript-application /absolute/path/to/app --json
```

## With a local model

Point Claude Code at colibri's
[Anthropic endpoint](api.md#anthropic-protocol-endpoint-v1messages) and start
it in the folder where you added REA:

```bash
export ANTHROPIC_BASE_URL=http://localhost:8000
export ANTHROPIC_API_KEY=local            # only enforced if you set COLI_API_KEY
export ANTHROPIC_MODEL=glm-5.2-colibri    # server.model_id in `coli status --json`
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1   # no telemetry or error reports either
claude
```

Other agents connect through the OpenAI-compatible endpoint the same way
([Connect a coding CLI or editor](api.md#connect-a-coding-cli-or-editor)).

Two things decide whether this is usable:

- **The engine must support tool calls.** Every engine in the
  [tool-calling matrix](api.md#tool-calling-support) does except Inkling and
  OLMoE.
- **The prompt is long.** Claude Code turns its tool search off when
  `ANTHROPIC_BASE_URL` points to a host other than Anthropic's, so every REA
  tool's name, description and input schema goes out with every request, on top
  of Claude Code's own system prompt. That is a long prefill before the first
  token: read the prefill warning in
  [api.md](api.md#connect-a-coding-cli-or-editor) before you connect. REA keeps
  its input schemas compact by default, so leave
  `REA_MCP_INPUT_SCHEMA_PROFILE` unset. A model that runs from RAM, on a GPU if
  there is one, is the practical choice; a model that streams its experts from
  the disk is too slow for an agent loop.

## Updating REA

The exact version is deliberate: REA's documentation asks for one exact version
in any registration that persists. To move to a newer release, look it up with
`npm view rea-agents version`, then remove the entry and add it again with the
new version:

```bash
claude mcp remove --scope local rea
claude mcp add --scope local rea -- npx -y rea-agents@<version> mcp
```

A registration made by `npx rea-agents setup` is refreshed with
`npx rea-agents@latest setup`.

## What you may analyze

Analyze software you own or are authorized to analyze. Commercial engines such
as IDA Pro, Hopper and JEB are obtained and used under their own licenses.
