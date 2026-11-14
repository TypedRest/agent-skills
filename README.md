# TypedRest agent skills

Agent skills that teach AI coding agents how to work with [TypedRest](https://typedrest.net/).

## Skills

### `typedrest-client`

Builds REST API client libraries with TypedRest in C#, Java, Kotlin or TypeScript.

- With an OpenAPI/Swagger document: sets up code generation (the Roslyn source generator for C#, the `typedrest-codegen` CLI for the other languages), checks and corrects what the generator infers, and wires regeneration into the build.
- Then adds hand-written code for what the generator does not cover: convenience methods, authentication, error handling, dependency injection.
- Without a spec: hand-writes the endpoint tree.
- Also covers project scaffolding and tests.

## Installation

With the [`skills` CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add TypedRest/agent-skills
```

Or copy the folders under `skills/` into your agent's skills directory, e.g. `.claude/skills/` in a project or `~/.claude/skills/` for Claude Code.
