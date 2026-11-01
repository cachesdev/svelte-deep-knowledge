# SvelteKit 3 reference

Verified 2026-11-01 against 3.1.1. Apply this file when the installed Kit version is 3.x or the user requests a Kit 3 migration. Stable Kit 2 guidance is in [SvelteKit 2 API reference](sveltekit-research.md).

SvelteKit 3.0.0 was released on October 1, 2026; 3.1.0 followed on October 14 and 3.1.1 on October 29. Remote functions and async Svelte remain experimental. The Svelte MCP and `svelte.dev/docs/kit` serve Kit 3 docs; `next.svelte.dev` redirects there. [Announcement](https://svelte.dev/blog/sveltekit-3-is-here), [October 2026](https://svelte.dev/blog/whats-new-in-svelte-october-2026), [changelog](https://github.com/sveltejs/kit/blob/main/packages/kit/CHANGELOG.md).

## Setup and migration

Minimum versions are Node 22.17, TypeScript 6, Svelte 5.57.1, Vite 8.0.12 and vite-plugin-svelte 7. Configure Kit through `sveltekit({...})` in Vite, with flattened Kit options alongside `compilerOptions`. Extend `$app/tsconfig` and specify project includes/excludes. [Migration guide](https://svelte.dev/docs/kit/migrating-to-sveltekit-3), [configuration](https://svelte.dev/docs/kit/@sveltejs-kit-vite).

```js
// vite.config.js
import { defineConfig } from 'vite';
import { sveltekit } from '@sveltejs/kit/vite';
import adapter from '@sveltejs/adapter-auto';

export default defineConfig({
  plugins: [
    sveltekit({
      adapter: adapter(),
      compilerOptions: { experimental: { async: true } },
      experimental: { remoteFunctions: true }
    })
  ]
});
```

`#lib` uses package.json subpath imports. Include extensions in imports such as `#lib/data.ts`; directory imports need the index filename. A typical mapping is:

```json
{
  "imports": {
    "#lib": "./src/lib/index.js",
    "#lib/*": "./src/lib/*"
  }
}
```

For an authorized migration, run `npx sv migrate sveltekit-3 --tasks all --confirm`, then review the generated TODOs. Upgrade to the latest 2.x first to get its deprecation warnings. Use `npx sv create` for a new app. [sv migrate](https://svelte.dev/docs/cli/sv-migrate).

Removed configuration options: `files.lib`, `experimental.handleRenderingErrors`, `experimental.instrumentation`, `experimental.tracing` (now top-level `tracing`), `vitePlugin` (pass vite-plugin-svelte options directly), `preloadStrategy`, `prerender.origin` (now `paths.origin`), and `csrf.checkOrigin` (now `csrf.trustedOrigins`). `output.linkHeaderPreload` is new and off by default. [Migration guide](https://svelte.dev/docs/kit/migrating-to-sveltekit-3#Configuration).

## Environment variables

Explicit env is standard in Kit 3. Export `variables = defineEnvVars(...)` from `src/env.ts`; import the helper from `@sveltejs/kit/env`. Variables default to private and dynamic; `public`, `static` and `description` match the Kit 2 opt-in. The `schema` option also accepts a function that returns a validated/transformed value or throws. [Environment variables](https://svelte.dev/docs/kit/environment-variables).

```ts
import { defineEnvVars } from '@sveltejs/kit/env';

export const variables = defineEnvVars({
  MAX_RESULTS: {
    schema: (raw) => {
      const value = Number(raw);
      if (!Number.isInteger(value) || value < 1) {
        throw new Error('MAX_RESULTS must be a positive integer');
      }
      return value;
    }
  }
});
```

Use `$app/env/private`, `$app/env/public` and `$app/env`. Kit 3 keeps `$env/*` and `$app/environment` as deprecated compatibility aliases. [Migration guide](https://svelte.dev/docs/kit/migrating-to-sveltekit-3#$env-...-deprecated).

## Navigation and snapshots

Use `goto(url, { shallow: true, state })` for shallow routing. Add `replace: true` to replace the entry and `persistState: true` to retain state after reload. Shallow changes invoke navigation hooks; inspect `navigation.shallow` when handling them. `reset: false` preserves focus and scroll together. `refreshAll()` takes no options and preserves page state; `invalidateAll()` is deprecated. Calling `invalidate` during a navigation no longer aborts it. [Navigation](https://svelte.dev/docs/kit/$app-navigation).

```js
await goto('/photos', {
  shallow: true,
  state: { selectedPhoto: photo.id },
  persistState: true
});
```

`preloadCode` accepts a route ID such as `/blog/[slug]`. Handle `preloadData`'s `{ type: 'error', status, error }` result. `goto` rejects destinations that do not resolve to an app route; use `window.location.href` for those. Navigation `delta` exists only for `popstate`. [Navigation API](https://svelte.dev/docs/kit/$app-navigation).

`updated.current` turns on after server data loads, remote function calls, tab focus or visibility changes, and a poll that defaults to one hour. Skew protection, such as Vercel's, can hide new deployments from the request-based checks. [Migration guide](https://svelte.dev/docs/kit/migrating-to-sveltekit-3#$app-state-updated-updates-automatically).

The `snapshot` helper belongs to `$app/navigation` and replaces `export const snapshot`. Call it during component initialization. It accepts capture/restore, an optional stable `id`, and optional `reset` for a new history entry without saved state. Values use the transport hook. [Snapshots](https://svelte.dev/docs/kit/snapshots).

```js
snapshot({
  id: 'search-filter',
  capture: () => filter,
  restore: (value) => { filter = value; },
  reset: () => { filter = ''; }
});
```

## Params and path types

Define matchers in `src/params.ts` with `defineParams` from `@sveltejs/kit/params`. A function returns the matched value or `undefined`; Standard Schema can transform a route string to a number, boolean or bigint. The output determines the typed param. Param and matcher names can contain hyphens. [Advanced routing](https://svelte.dev/docs/kit/advanced-routing#Matching).

```ts
import { defineParams } from '@sveltejs/kit/params';
import * as v from 'valibot';

export const params = defineParams({
  integer: v.pipe(v.string(), v.toNumber(), v.integer())
});
```

Use `[id=integer]` in the route. Generated names include `Path`, `AssetPath`, `PageRouteId` and `EndpointRouteId`. `Path` and `AssetPath` omit a leading slash; route IDs retain their own syntax. `RouteId` includes only routes with a `+page` or `+server`. Resolve links with `$app/paths` and check the exact accepted type. [Paths](https://svelte.dev/docs/kit/$app-paths), [generated types](https://svelte.dev/docs/kit/$app-types).

## Errors

`handleError` receives every error with a `kind`: `'app'` for `error(...)`, `'framework'` for Kit errors such as 404, `'validation'` for invalid remote function arguments (server only, with `issues`), and `'unknown'` for thrown values. Return `status` and `message` only to override the defaults. `kind: 'validation'` replaces `handleValidationError`. [Hooks](https://svelte.dev/docs/kit/hooks#handleError), [errors](https://svelte.dev/docs/kit/errors).

Rendering errors go through `handleError` to the nearest `+error.svelte`, which Kit wraps in a boundary. An async `handleError` in `hooks.client.ts` needs `compilerOptions.experimental.async`. `App.Error` includes `status`; use `error(status, message, details)` for structured errors. Enhanced form action responses use the status passed to `fail(...)`.

External redirects require `redirect(status, url, { external: true })` or an allowlist of origins. Use an allowlist where destinations come from user input. Enhanced cross-page form actions navigate to the action page for success and failure. [Migration guide](https://svelte.dev/docs/kit/migrating-to-sveltekit-3#Miscellaneous).

## Remote functions

Remote query code cannot access `event.url`, `event.params` or `event.route`; pass route-dependent inputs explicitly. Every submitted form control must come from `form.fields.name.as(...)`; a hand-written `name` makes Kit reject the submission. Remote resource `error` properties are typed `App.Error | undefined`. [Remote functions](https://svelte.dev/docs/kit/remote-functions).

Radio and checkbox options take the checked state as a third argument, so the input resets to it after a submission. Fields expose `touched()` and `dirty()`. `validate({ all: true })` replaces `validate({ includeUntouched: true })`; the 3.1 docs page still shows the old option name, while the 3.1.1 types use `all`. `form.submitted` resets with the form.

```svelte
{#each ['public', 'private'] as option}
  <label>
    <input {...settings.fields.visibility.as('radio', option, current.visibility === option)} />
    {option}
  </label>
{/each}
```

The server must handle every client-requested refresh within the `requested` limit: refresh it, set it, reconnect it, or ignore it. An unhandled request puts that query into an error state on the client. [Client-requested refreshes](https://svelte.dev/docs/kit/remote-functions#Single-flight-mutations-Client-requested-refreshes).

```js
for (const { arg, query, ignore } of requested(getPosts, 10)) {
  if (arg.filter === 'author:santa') {
    void query.refresh();
  } else {
    ignore();
  }
}
```

`requested(getPosts, 10).ignoreAll()` ignores every entry within the limit. Entries past the limit still fail on the client.

## Imports and responses

| Types or helpers | Kit 3 import |
| --- | --- |
| Remote function types (`RemoteQuery`, `RemoteForm`, `RemoteCommand`) | `$app/server` |
| Navigation types (`Navigation`, `GotoOptions`, `AfterNavigate`) | `$app/navigation` |
| `Page`, `ReadonlyURL`, `ReadonlyURLSearchParams` | `$app/state` |
| `ActionResult`, `SubmitFunction` | `$app/forms` |
| Hooks types (`Handle`, `HandleServerError`) | `@sveltejs/kit/hooks` |
| Environment types, `defineEnvVars` | `@sveltejs/kit/env` |
| `defineParams` and param types | `@sveltejs/kit/params` |

`RequestEvent`, `Cookies`, `error`, `redirect`, and `isHttpError` stay in `@sveltejs/kit`. `json()` and `text()` are deprecated; use `Response.json(...)` and `new Response(...)`. 204 responses have no body, and `handle`'s `resolve` always returns a `Promise`. `+server` files can export a `QUERY` handler. [Migration guide](https://svelte.dev/docs/kit/migrating-to-sveltekit-3#@sveltejs-kit).

`page.url` is a `ReadonlyURL`; copy it with `new URL(page.url.href)` before editing. `$app/stores` is removed. Clicking a link to the current URL runs `refreshAll()`. `data-sveltekit-*` attributes take `false` where Kit 2 took `'off'`, and `data-sveltekit-reset` replaces `data-sveltekit-noscroll` and `data-sveltekit-keepfocus`.

## Security, cookies and server-only modules

Cookie names must be ASCII (`cookie` v2), and the cookie `path` defaults to `'/'`. Cross-origin form submissions need a `Content-Type` header. Query parameters starting with `x-sveltekit-` are rejected. Kit 3.1 accepts the `report-sha256`, `report-sha384` and `report-sha512` CSP keywords.

A `server` filename segment (`db.server.ts`, `server.ts`) or any `server` directory outside `src/routes` and the assets directory marks a module server-only. A `remote` filename segment marks a remote module, and remote modules cannot live in a `server` directory. [Server-only modules](https://svelte.dev/docs/kit/server-only-modules).

## Service workers and observability

Service workers use `$app/env`, `$app/paths` and `$app/manifest` for version, URLs and build metadata. Import typed `self` from `$app/service-worker`. Give the worker its own tsconfig extending `$app/tsconfig/service-worker`, and exclude it from the app tsconfig. The worker is registered as a module. [Service workers](https://svelte.dev/docs/kit/service-workers).

`src/instrumentation.server.ts` loads automatically, after env vars are populated. Enable OpenTelemetry spans with top-level `tracing: { server: true }`. Kit 3.1 adds a `sveltekit.endpoint` span for `+server` handlers. [Observability](https://svelte.dev/docs/kit/observability).

## Adapters

All first-party adapters require Kit 3. Review `paths.origin`, cookie defaults and adapter versions during migration. [Adapter migration](https://svelte.dev/docs/kit/migrating-to-sveltekit-3#Adapters).

- **adapter-cloudflare 8** removes `platform`. Import `env` and `waitUntil` from `cloudflare:workers`, read `request.cf`, and use the global `caches`. Run `wrangler types` (wrangler 4.67+) for the types.
- **adapter-node 6** bundles with rolldown and serves only static files present at build time, with content-hash ETags, for `GET` and `HEAD`. Set `paths.origin` in place of the `ORIGIN` environment variable.
- **adapter-netlify 7** uses the Netlify Frameworks API and reads the publish directory from an adapter option. `split` and `edge` can be combined.
- **adapter-vercel** drops the edge runtime.
- **adapter-auto** supports zero-config Render deployments.
- **adapter-bun** builds a standalone `Bun.serve` server. It needs Bun 1.4+, and the build must run in Bun.

```js
// src/routes/api/+server.js
import { env, waitUntil } from 'cloudflare:workers';

export async function GET({ request }) {
  const value = await env.KV.get('key');
  waitUntil(env.KV.put('last-country', request.cf?.country ?? 'unknown'));
  return Response.json({ value });
}
```

Adapter authors: adapters can add `pre`/`post` Vite plugins and receive the Svelte config, override `getRequest`/`setResponse` during `vite dev` and `vite preview`, and use `applyReroute` for split serverless deployments. `builder.generateServerInstance` and `builder.manifest` replace `builder.generateManifest`; `builder.createEntries` and the public `Server` constructor are removed. [Writing adapters](https://svelte.dev/docs/kit/writing-adapters).
