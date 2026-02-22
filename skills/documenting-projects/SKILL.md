---
name: documenting-projects
description: Use when analyzing a codebase to generate documentation — developer docs (architecture, modules, diagrams) or user guides (installation, CLI reference, usage workflows). Use when existing documentation is missing, outdated, or superficial.
---

# Documenting Projects

## Overview

**Read first, document only what you verified.** Never write documentation about code you haven't read. Every claim must trace back to a specific file and line number.

## When to Use

- Generating project documentation from scratch
- Updating outdated docs after major changes
- Creating onboarding guides for new developers
- Documenting architecture, APIs, or internal modules
- Writing installation guides, CLI references, or user-facing usage docs

**When NOT to use:** Marketing copy, API docs for external consumers that should use OpenAPI/Swagger

## Core Rule: VERIFY BEFORE DOCUMENT

```
NEVER document what you haven't read in the actual code.
```

**This means:**
- Read every file before documenting it
- If you can't read a file, say "not verified" instead of guessing
- Never invent API responses, model fields, or config values
- Never assume a file's content from its name alone

**Violations:**
- Writing "User model has `username`, `email` fields" without reading `user.py` → FABRICATION
- Documenting API response JSON without reading the endpoint handler → FABRICATION
- Listing CLI commands without reading the actual CLI source or `--help` output → FABRICATION

## Process

### Phase 0: CLASSIFY — Determine Documentation Type

Before reading any code, determine which documentation track is needed:

| Track | When to Use | Examples |
|-------|-------------|---------|
| **Developer Docs** | Reader will read/modify the code | Architecture overview, module docs, class diagrams, data flow, onboarding guide |
| **User Guide** | Reader will use the software as-is | Installation guide, CLI reference, configuration guide, troubleshooting |
| **Both** | Project needs both perspectives | Open-source tool needing contributor docs + user docs |

**How to decide:**
- User explicitly asks for "user guide", "installation docs", "CLI reference" → **User Guide**
- User asks for "architecture", "onboarding", "module docs", "class diagram" → **Developer Docs**
- User asks to "document this project" (generic) → Ask which track, or default to **Developer Docs**
- User asks for "full documentation" → **Both** (write them as separate documents)

### Phase 1: SCAN (Breadth-First Discovery)

Map the project structure without documenting content yet.

```
1. Read project root files: README, package manager configs, Makefile, Dockerfile, .env.example
2. List directory tree (2 levels deep)
3. Identify project type and tech stack from actual config files
4. Note entry points (main, wsgi, index, cli)
5. Map dependency graph from import statements
```

**Output:** Mental model of project shape. No docs yet.

### Phase 2: DEEP-READ (Depth-First Analysis)

Read files in priority order:

```
Priority 1: Entry points (main, wsgi, __init__.py, cli)
Priority 2: Config files (config.py, settings, .env.example)
Priority 3: Core models/schemas (data definitions)
Priority 4: Business logic (services, handlers, controllers)
Priority 5: Infrastructure (middleware, utils, helpers)
Priority 6: Build/deploy (Dockerfile, CI, Makefile)
```

For each module, note: public interface, dependencies, side effects, data flow, WHY.

### Phase 3: WRITE — Follow the Track Guide

Based on the classification from Phase 0, read the appropriate guide and follow it:

- **Developer Docs** → Read `skills/documenting-projects/developer-docs-guide.md` and follow its instructions
- **User Guide** → Read `skills/documenting-projects/user-guide.md` and follow its instructions
- **Both** → Read both files; produce separate documents for each audience

## Project-Adaptive Document Structure

Choose sections based on what the project actually needs:

| Project Type | Essential Sections | Skip |
|-------------|-------------------|------|
| **Web API** | Architecture, API Reference, Data Models, Auth Flow, Error Codes | Pipeline/DAG docs |
| **Data Pipeline** | Data Flow Diagram, Transform Chain, Schema, Scheduling, Monitoring | API endpoints |
| **CLI Tool** | Commands Reference, Config, I/O Format, Exit Codes | Architecture layers |
| **Library/SDK** | Public API, Usage Examples, Extension Points | Deployment |
| **Monorepo** | Package Map, Inter-package Dependencies, Build Order | Single-service setup |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Documenting code you haven't read | Read file first, cite line numbers |
| Inventing API responses | Read the route handler, extract actual response shape |
| Same template for every project | Adapt sections to project type |
| Listing features without context | Explain WHY each design decision was made |
| Copy-pasting entire source files | Extract key patterns, reference file:line for details |
| Environment vars as simple list | Include required/default/impact columns |
| Flat module listing | Show dependency tree and data flow |
| Missing error documentation | Trace exception hierarchy and handling |
| Mixing developer and user audiences | Separate into distinct documents with appropriate depth |

## Handling Unverified Content

When you can't read some files, separate verified from unverified:

```markdown
## Verified Components
(detailed docs with file:line references)

## Components Requiring Verification
| Module | File | What's Needed |
|--------|------|---------------|
| Auth endpoints | api/auth.py | Endpoint signatures, request/response formats |
| User model | models/user.py | Field definitions, relationships |
```

**Never scatter `[Not Verified]` throughout the document.** Consolidate unverified items into a dedicated section at the end.

## Documentation Quality Checklist

### Common (Both Tracks)

- [ ] Every documented fact traces to a specific file:line
- [ ] No invented/assumed content (model fields, API responses, commands)
- [ ] Document structure matches project type (not generic template)
- [ ] Quick start commands are verified against actual config files
- [ ] Unverified content consolidated in dedicated section (not scattered)

### Developer Docs Track

- [ ] Architecture explains WHY, not just WHAT
- [ ] Module dependency map shows who calls whom
- [ ] Data flow documented for key operations
- [ ] Environment variables have required/default/impact
- [ ] Error handling patterns documented
- [ ] Mermaid diagrams (if included) render valid syntax

### User Guide Track

- [ ] Written from user perspective, not developer perspective
- [ ] Installation steps are copy-paste runnable
- [ ] CLI commands verified against actual `--help` output
- [ ] All examples use realistic, working values
- [ ] Troubleshooting covers actual error messages from source code
- [ ] No internal implementation details leaked to user docs
