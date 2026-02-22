# User Guide Track

This guide is read by the documenting-projects skill after Phase 2 (DEEP-READ) when the documentation target is **User Guide**. All content here assumes you have already scanned and read the codebase.

## What User Guides Cover

User guide documentation helps someone who will **use the software without reading its source code**. Focus on:

- How to install and set up the software
- What commands/features are available and how to use them
- How to configure behavior
- What to do when things go wrong

## Writing Principles

1. **User perspective, not developer perspective** — Describe what the software *does*, not how it's *implemented*
2. **Action-oriented** — Lead with what the user can do, not internal architecture
3. **Copy-paste ready** — Every command and example should be runnable as written
4. **Minimal jargon** — If a technical term is unavoidable, define it on first use
5. **Verify everything** — Run commands or read source to confirm behavior. The VERIFY BEFORE DOCUMENT rule applies equally here

## Installation Guide

Structure installation docs in this order:

### 1. Prerequisites

List what must already be installed. Read these from actual config files (e.g., `engines` in `package.json`, `python_requires` in `pyproject.toml`, Dockerfile `FROM` image).

```markdown
## Prerequisites

- Python 3.10 or later
- pip or uv package manager
- PostgreSQL 14+ (for database)
```

### 2. Installation Commands

Provide verified install commands. Read from `README.md`, `Makefile`, `pyproject.toml`, `package.json`, or `Dockerfile`.

```markdown
## Installation

```bash
pip install my-tool
```

Or from source:

```bash
git clone https://github.com/org/my-tool.git
cd my-tool
pip install -e .
```
```

**IMPORTANT:** Only write install commands you have verified in the project's actual config files. If the project uses `uv`, don't write `pip install`. If it uses `pnpm`, don't write `npm install`.

### 3. Verification

Show how to confirm successful installation.

```markdown
## Verify Installation

```bash
my-tool --version
# Expected output: my-tool 1.2.3
```
```

Read the actual version string from `__version__`, `version` field in config, or `Cargo.toml`.

## CLI Command Reference

Build the CLI reference by reading the actual CLI source code (argument parsers, command definitions) and `--help` output.

### Command Table

```markdown
## Commands

| Command | Description | Example |
|---------|-------------|---------|
| `my-tool init` | Initialize a new project in current directory | `my-tool init --template basic` |
| `my-tool build` | Build the project | `my-tool build --release` |
| `my-tool serve` | Start development server | `my-tool serve --port 8080` |
```

### Options / Flags Table

For each command (or globally), document options:

```markdown
### `my-tool build` Options

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--release` | bool | `false` | Enable optimized release build |
| `--output`, `-o` | string | `./dist` | Output directory path |
| `--jobs`, `-j` | int | CPU count | Parallel build jobs |
| `--verbose`, `-v` | bool | `false` | Show detailed build output |
```

**Source these from:** `argparse` definitions, `clap` structs, `commander` declarations, or the actual `--help` output. Do NOT invent flags.

### Exit Codes

If the program defines exit codes, document them:

```markdown
### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | General error |
| `2` | Invalid arguments |
| `78` | Configuration error |
```

Source these from the actual code (search for `sys.exit`, `process.exit`, `std::process::exit`, or exit code constants).

## Configuration Guide

### Config File Location and Format

```markdown
## Configuration

Config file location: `~/.config/my-tool/config.toml`

The tool searches for configuration in this order:
1. Command-line flags (highest priority)
2. Environment variables
3. Config file
4. Built-in defaults
```

Read the actual config loading logic to determine precedence order and file paths.

### Environment Variables (User Perspective)

Unlike developer docs (which document all env vars with implementation details), user guides focus on **what changes for the user**:

```markdown
## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `MY_TOOL_CONFIG` | Override config file path | `MY_TOOL_CONFIG=./custom.toml` |
| `MY_TOOL_LOG_LEVEL` | Set logging verbosity: `debug`, `info`, `warn`, `error` | `MY_TOOL_LOG_LEVEL=debug` |
| `NO_COLOR` | Disable colored output | `NO_COLOR=1` |
```

## Usage Scenarios / Workflows

Document common workflows as step-by-step guides:

```markdown
## Common Workflows

### Starting a New Project

1. Create and initialize:
   ```bash
   mkdir my-project && cd my-project
   my-tool init --template web
   ```

2. Configure your settings:
   ```bash
   my-tool config set database.url "postgres://localhost/mydb"
   ```

3. Start development:
   ```bash
   my-tool serve
   ```
   Open `http://localhost:3000` in your browser.

### Deploying to Production

1. Build optimized output:
   ```bash
   my-tool build --release
   ```

2. The output is in `./dist/`. Deploy it to your server:
   ```bash
   scp -r ./dist/ user@server:/var/www/my-project/
   ```
```

**Guidelines:**
- Base workflows on actual project capabilities verified in code
- Use realistic values in examples (not `foo`, `bar`, `example.com`)
- Show expected output where it helps confirm success

## Troubleshooting / FAQ

Build a troubleshooting section from actual error messages in the source code (search for error strings, exception messages, log output).

```markdown
## Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `Error: Config file not found` | No config file at expected path | Run `my-tool init` or create `~/.config/my-tool/config.toml` |
| `Error: Port 3000 already in use` | Another process is using the port | Use `--port 3001` or stop the other process |
| `Error: Database connection refused` | Database is not running or URL is wrong | Check `DATABASE_URL` and ensure the database is running |
```

**How to find real error messages:**
- Search for `raise`, `Error(`, `panic!`, `log.error`, `console.error` in source
- Read validation functions for input error messages
- Check exit code handling for CLI error messages

## User Guide Quality Checklist

Before delivering a user guide, verify:

- [ ] Written entirely from user perspective (no "this module calls..." language)
- [ ] Installation steps are copy-paste runnable and verified against project config
- [ ] CLI commands match actual `--help` output or argument parser definitions
- [ ] All examples use realistic, working values
- [ ] Config file paths and formats verified in source code
- [ ] Environment variables sourced from actual code, not guessed
- [ ] Troubleshooting errors are real error messages found in source
- [ ] No internal implementation details exposed to users
- [ ] Prerequisites list verified from actual dependency/config files
- [ ] Unverified sections clearly marked and consolidated at the end
