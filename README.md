# svelte-deep-knowledge

[![skills.sh](https://skills.sh/b/cachesdev/svelte-deep-knowledge)](https://skills.sh/cachesdev/svelte-deep-knowledge)

An agent skill for Svelte 5 and SvelteKit. It lists the APIs coding agents most often get wrong or have never seen, and has the agent check them against the official Svelte docs before it writes code.

## Install

```bash
npx skills add cachesdev/svelte-deep-knowledge
```

This installs the skill into the current project. Add `-g` to install it for every project.

The skill fetches docs and runs the autofixer through the [Svelte MCP server](https://svelte.dev/docs/ai/mcp). Set that up first.

## What it covers

- Svelte 5 runes, `await` in components, `<svelte:boundary>`, attachments, and declaration tags
- SvelteKit 3 configuration in `vite.config.ts`, `#lib`, `$app/env`, navigation, error handling, and adapter changes
- Remote functions in SvelteKit 2.27+ and 3
- SvelteKit 2 projects, in a separate reference

Entries list the version that introduced them. The agent reads the project's installed versions first and applies only the guidance that matches.

## Versions

Last checked on 2026-11-01 against Svelte 5.57.3, SvelteKit 3.1.1, and SvelteKit 2.70.3. Async Svelte and remote functions are still experimental, and their APIs can change in minor releases.

## License

[MIT](LICENSE)
