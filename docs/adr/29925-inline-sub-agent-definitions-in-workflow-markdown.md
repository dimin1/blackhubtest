# ADR-29925: Inline Sub-Agent Definitions in Workflow Markdown Files

**Date**: 2026-05-03
**Status**: Draft
**Deciders**: Unknown [TODO: verify — identify the team members who designed the inline sub-agent feature]

---

## Part 1 — Narrative (Human-Friendly)

### Context

The `gh-aw` system allows users to define agentic workflows as Markdown files (`.md`). As multi-agent orchestration use-cases grew, workflows increasingly needed to delegate sub-tasks to secondary agents with their own prompts and model configurations. Before this feature, each agent had to live in a separate file, requiring workflow authors to manage multiple files and cross-file references even for tightly related agent pairs. The `pkg/parser` package, which processes these workflow files for both the CLI binary and WebAssembly targets, needed a mechanism to support co-located agent definitions without breaking existing single-agent workflows.

### Decision

We will allow workflow markdown files to embed inline sub-agent definitions using `## agent: \`name\`` level-2 headings as delimiters. The `pkg/parser` package will provide `ExtractInlineSubAgents` to split a workflow file into the primary workflow body and a list of `InlineSubAgent` structs, each carrying an optional frontmatter block (only `description` and `model` fields are valid) and a prompt body. Engine-specific directory layout and file extension for persisted sub-agent files are resolved via `GetEngineSubAgentDir` and `GetEngineSubAgentExt`, keeping engine details out of the parser.

### Alternatives Considered

#### Alternative 1: Separate Files per Sub-Agent

Each sub-agent lives in its own `.md` file, with the primary workflow referencing it by path or name. This is the pre-existing convention for multi-file workflows. It was not chosen for tightly coupled sub-agents because it forces authors to manage multiple files for what is logically a single workflow, increases filesystem scatter, and complicates sharing in single-file snippets and examples.

#### Alternative 2: YAML Front-Matter Block for Sub-Agents

Sub-agent definitions could be encoded as a structured YAML list inside the primary workflow's frontmatter block. This was not chosen because it conflates workflow configuration (frontmatter) with prompt content (markdown body), and YAML is poorly suited for multi-paragraph natural language prompts that sub-agents typically require.

### Consequences

#### Positive
- A single `.md` file can express a complete multi-agent interaction, improving authoring ergonomics and portability.
- The `## agent: \`name\`` delimiter is valid Markdown, so existing renderers display sub-agent sections as readable headings rather than raw metadata.
- Validation helpers (`ValidateInlineSubAgentsFrontmatter`, `ValidateInlineSubAgentsInBody`) catch invalid frontmatter fields at compile time, preventing silent misconfiguration.

#### Negative
- Files that use inline sub-agents are harder to diff and review in pull requests because unrelated agent prompts are co-located.
- The `## agent:` heading syntax is now a reserved parser token; workflow authors cannot use this exact heading pattern for regular content without triggering sub-agent extraction.
- Only `description` and `model` are valid frontmatter fields for inline sub-agents; any future expansion of sub-agent configuration requires parser changes and a backward-compatibility decision.

#### Neutral
- Engine-specific persisted layout is abstracted via `GetEngineSubAgentDir`/`GetEngineSubAgentExt`, so adding a new engine does not require changes to the inline sub-agent extraction logic.
- The feature was first exposed in the public `pkg/parser` API through README documentation as part of a spec-drift remediation pass; ADR creation is retroactive. [TODO: verify — confirm the implementation commit date and original design discussion]

---

## Part 2 — Normative Specification (RFC 2119)

> The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this section are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### Inline Sub-Agent Syntax

1. An inline sub-agent section **MUST** begin with a level-2 heading matching the pattern `## agent: \`<name>\`` where `<name>` is the sub-agent identifier.
2. An inline sub-agent section **MUST NOT** begin with any other heading level or heading pattern.
3. The primary workflow content **MUST** comprise all content before the first `## agent:` heading; `ExtractInlineSubAgents` **MUST** return this slice as `mainMarkdown`.
4. Each inline sub-agent **MAY** include a YAML frontmatter block immediately following its heading delimiter.
5. Inline sub-agent frontmatter **MUST NOT** contain fields other than `description` and `model`; `ValidateInlineSubAgentsFrontmatter` **MUST** emit an advisory warning string for each invalid field encountered.

### Engine-Specific Persistence

1. Callers that persist inline sub-agents to disk **MUST** use `GetEngineSubAgentDir(engineID)` to resolve the target directory.
2. Callers that persist inline sub-agents to disk **MUST** use `GetEngineSubAgentExt(engineID)` to resolve the file extension.
3. Implementations **MUST NOT** hard-code engine-specific directory names or extensions outside of `GetEngineSubAgentDir` and `GetEngineSubAgentExt`.

### Conformance

An implementation is considered conformant with this ADR if it satisfies all **MUST** and **MUST NOT** requirements above. Failure to meet any **MUST** or **MUST NOT** requirement constitutes non-conformance.

---

*ADR created by [adr-writer agent]. Review and finalize before changing status from Draft to Accepted.*
