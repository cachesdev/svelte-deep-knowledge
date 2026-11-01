# SvelteKit 2 API reference

Verified 2026-11-01. Kit 2.70.3, released August 18, 2026, is the final 2.x release; no 2.x patch has shipped since SvelteKit 3.0.0 on October 1. Use the examples with the stated feature flags and minimum versions.

## Coverage and release status

Reviewed the complete stable package changelog, 368 entries from 1.0.0 through 2.70.3, against the official MCP sections for remote functions, configuration, environment variables, navigation, paths, hooks, loading data, observability and public types. Also scanned the linked pre-1.0 archive, 607 entries back to 0.0.14, and the version-3 changelog. Reviewed the June through September 2026 official posts and the SvelteKit 3 RC announcement. Historical entries identify obsolete APIs; current docs determine present usage. [Stable changelog](https://github.com/sveltejs/kit/blob/main/packages/kit/CHANGELOG.md), [pre-1.0 archive](https://github.com/sveltejs/kit/blob/main/packages/kit/CHANGELOG-pre-1.md).

The Svelte MCP and `svelte.dev/docs/kit` now serve Kit 3 docs. For Kit 2 behavior, read the `version-2` branch: [Kit 2 docs source](https://github.com/sveltejs/kit/tree/version-2/documentation/docs) and [Kit 2 changelog](https://github.com/sveltejs/kit/blob/version-2/packages/kit/CHANGELOG.md). Links below to `svelte.dev/docs/kit` show Kit 3 pages; compare signatures with the installed Kit 2 types. [Kit 3 reference](sveltekit-3.md).

## Version and API checks

- In Kit 2.70+, import `defineEnvVars` from `@sveltejs/kit/env`. Use the `schema` property for a Standard Schema validator. Export `variables` from `src/env.ts`. Variables default to private and dynamic; `public: true` and `static: true` are separate choices. Kit 2 needs `experimental.explicitEnvironmentVariables`. [Environment variables](https://svelte.dev/docs/kit/environment-variables), [helper reference](https://svelte.dev/docs/kit/@sveltejs-kit-env).
- Remote functions remain experimental. The official guide recommends `await` and explicitly supports `loading`, `error` and `current` as another way to consume a query. Commands can run outside render, including imperative callbacks; their use is broader than event handlers. Prefer a remote `form` when there is a natural form submission. [Remote functions](https://svelte.dev/docs/kit/remote-functions).
- `load` remains a supported routing and data-loading API. Universal `load` can await remote queries. Keep architectural preferences separate from claims about API deprecation. [Loading data](https://svelte.dev/docs/kit/load).
- Kit 2.62 permits flattened configuration in `sveltekit({...})`; providing plugin config causes SvelteKit to ignore `svelte.config.js`. Current docs also permit `svelte.config.js` for Kit 2. Follow the project's actual configuration. [Configuration](https://github.com/sveltejs/kit/blob/version-2/documentation/docs/98-reference/50-configuration.md).
- `getRequestEvent` arrived in 2.20, before remote functions arrived in 2.27. Phrase these as separate milestones. Its `route`, `params` and `url` have special remote-function semantics. [Server API](https://svelte.dev/docs/kit/$app-server), [request-event guidance](https://svelte.dev/docs/kit/remote-functions#Using-getRequestEvent).

## Practical additions for Kit 2

### Remote forms and queries

- `form.submit()` is available from 2.61 and resolves to a validation-success boolean. The custom `enhance` callback receives a form instance; it controls submission and reset. Kit 2.68 exports `RemoteFormEnhanceCallback` and `RemoteFormEnhanceInstance` from `@sveltejs/kit`. Kit 2.69 adds `submitted`, which records whether the form has been submitted at least once. `pending` counts active submissions. These values do not mean the same thing. [RemoteForm types](https://svelte.dev/docs/kit/@sveltejs-kit#RemoteForm), [enhance callback types](https://svelte.dev/docs/kit/@sveltejs-kit#RemoteFormEnhanceCallback).
- `field.as(type, defaultValue)` controls both the rendered value and the reset value. Unchecked checkboxes are absent from `FormData`, so boolean schemas need an optional/default case. Use `field.as('submit', value)` for multiple submit buttons; the old `buttonProps` API was removed in 2.50. Use `enctype="multipart/form-data"` for file inputs. Commands also accept `File` values from 2.64. [Fields](https://svelte.dev/docs/kit/remote-functions#form-Fields), [July 2026](https://svelte.dev/blog/whats-new-in-svelte-july-2026).
- `invalid` is imported from `@sveltejs/kit`; the field-issue builder is the handler's second argument. `invalid(issue.field('message'))` fails validation for that field. Avoid wrapping it in a catch that swallows the thrown validation error. [Validation helper](https://svelte.dev/docs/kit/@sveltejs-kit#invalid).
- `query.batch`'s handler receives an argument array and returns an `(input, index) => result` function. All requests within one macrotask are batched. Cache keys normalize object, Map and Set ordering; use arrays when order is meaningful. Dynamic queries cannot run in an entirely prerendered page; use `prerender` for static data. [Batch queries](https://svelte.dev/docs/kit/remote-functions#query.batch), [query arguments](https://svelte.dev/docs/kit/remote-functions#query-Query-arguments).
- Since 2.61, `await query()` works in render and imperative contexts, sharing the active cache. `.run()` was removed. Cache retention follows active consumers, so it is not a persistent general-purpose cache. [Deduplication](https://svelte.dev/docs/kit/remote-functions#query-Deduplication).
- For client-selected refreshes, use `submit().updates(...)` and explicitly accept a bounded list with `requested(getItems, limit)`. Its entries contain transformed `arg` plus a `query` bound to the original cache key. Use that bound query for `set`/`refresh`. `requested(getItems, limit).refreshAll()` is the shorthand. [Client-requested refreshes](https://svelte.dev/docs/kit/remote-functions#Single-flight-mutations-Client-requested-refreshes).
- A successful form normally refreshes active loads and queries; a command does not automatically refresh them. Server-triggered refreshes or `set` values travel with the mutation response. Server-side refresh completion indicates scheduling, not completion of the query work. Queries can refresh other queries from 2.65. [Single-flight mutations](https://svelte.dev/docs/kit/remote-functions#Single-flight-mutations).
- `query.live` arrived in 2.59; instances became async-iterable in 2.61. SSR consumes its first value and closes the iterator. Active browser consumers share a connection, reconnect with backoff, and receive the latest pending value if they consume slowly. It has `connected` and `reconnect`, with no `refresh`. Keep `no-store` responses out of service-worker caches. [Live queries](https://svelte.dev/docs/kit/remote-functions#query.live).

### Routing, hooks and rendering

- `PageProps` and `LayoutProps` are generated in `./$types` from 2.16. A typed `params` prop is available from 2.24. Kit 2.55 narrows route params using matcher type guards. Use generated types before inventing component interfaces. [Types](https://svelte.dev/docs/kit/types), [routing](https://svelte.dev/docs/kit/routing).
- `resolve` and `asset` arrived in 2.26; `match` in 2.52. `resolve` supports query strings and fragments from 2.54. It handles app paths, and rejects external URLs. [Paths](https://svelte.dev/docs/kit/$app-paths).
- Navigation targets expose `scroll` from 2.51. Before navigation, destination scroll can be unknown; `afterNavigate` reports the applied position. [NavigationTarget](https://svelte.dev/docs/kit/@sveltejs-kit#NavigationTarget).
- `reroute` is universal and runs before route selection. It maps a URL to a route without changing the address bar. It can be asynchronous from 2.18 and gets enhanced `fetch` from 2.19. Keep it deterministic and fast because it can delay navigation. `init` in server/client hooks runs once; async client initialization delays hydration. [Hooks](https://svelte.dev/docs/kit/hooks).
- Kit 2.31 adds OpenTelemetry tracing and `instrumentation.server.ts`. Enable the Kit 2 tracing/instrumentation options and check adapter support. Kit 3 changes these flags. [Observability](https://svelte.dev/docs/kit/observability).
- SSR error-boundary integration arrived in Kit 2.54. Enable `experimental.handleRenderingErrors` and inspect how nearest `+error.svelte` pages and `handleError` work before wrapping every route manually. [Configuration](https://github.com/sveltejs/kit/blob/version-2/documentation/docs/98-reference/50-configuration.md).
- `prerender.handleInvalidUrl`, from 2.67, controls malformed links found by the crawler. It accepts `fail`, `warn`, `ignore` or a handler receiving `href`, `referrer` and `message`. [Prerender configuration](https://github.com/sveltejs/kit/blob/version-2/documentation/docs/98-reference/50-configuration.md).

## Release-sensitive maintenance

Kit 2.65.2 prevents shared caches from caching personalized remote responses and preserves transport error statuses. Kit 2.66 fixes live-query reconnect/iteration and invalidation deadlocks. Kit 2.70.2 fixes pathological Accept-header parsing; 2.70.3 fixes eager `$app/state` dependency reads. Check installed patch versions when diagnosing those symptoms. [Stable changelog](https://github.com/sveltejs/kit/blob/main/packages/kit/CHANGELOG.md).

Async Svelte and remote functions remain experimental in Kit 3. Kit 3 migration guidance belongs behind an explicit major-version check. [SvelteKit 3 announcement](https://svelte.dev/blog/sveltekit-3-is-here).

## Usage examples

### Explicit environment variables, Kit 2.70+

Set `experimental.explicitEnvironmentVariables: true` in the Kit options. This example uses Standard Schema validation and keeps the token private. [Environment variables](https://svelte.dev/docs/kit/environment-variables).

```ts
// src/env.ts
import { defineEnvVars } from '@sveltejs/kit/env';
import * as v from 'valibot';

export const variables = defineEnvVars({
  API_TOKEN: { schema: v.pipe(v.string(), v.nonEmpty()) },
  SITE_NAME: { public: true, static: true, schema: v.string() }
});
```

Application imports are `import { API_TOKEN } from '$app/env/private'` on the server and `import { SITE_NAME } from '$app/env/public'` where the public value is needed. `browser`, `building`, `dev` and `version` come from `$app/env` in this mode.

### Custom remote form submission, Kit 2.69+

Given a remote form named `saveProfile` with a string `name` field and `{ saved: true }` result, its `enhance` callback controls submit and reset. `submitted` tracks an attempt; `pending` tracks work in progress; `result` contains the handler's return value. Transport/server failures can still throw and need the app's error reporting. [RemoteForm](https://svelte.dev/docs/kit/@sveltejs-kit#RemoteForm).

```svelte
<script>
  import { saveProfile } from './profile.remote';
</script>

<form {...saveProfile.enhance(async (form) => {
  if (await form.submit()) form.element.reset();
})}>
  <label>Name <input {...saveProfile.fields.name.as('text')} /></label>
  <button disabled={saveProfile.pending > 0}>Save</button>
  {#if saveProfile.submitted}
    {#each saveProfile.fields.name.issues() ?? [] as issue (issue)}
      <p>{issue.message}</p>
    {/each}
  {/if}
</form>
```

### Live query consumption, Kit 2.61+

Given a `query.live` export named `getStatus` that yields strings, the boundary handles its initial value and errors. `connected` reports stream connection status. Consumers receive the latest status value. [Live queries](https://svelte.dev/docs/kit/remote-functions#query.live).

```svelte
<script>
  import { getStatus } from './status.remote';
  const status = getStatus();
</script>

<svelte:boundary>
  <p>{await status}</p>
  <button onclick={() => status.reconnect()} disabled={status.connected}>
    Reconnect
  </button>
  {#snippet pending()}<p>Loading status...</p>{/snippet}
  {#snippet failed(error, reset)}
    <p>Status is unavailable.</p>
    <button onclick={reset}>Try again</button>
  {/snippet}
</svelte:boundary>
```

### Refresh scope

In Kit 2, `invalidateAll()` reruns the active page's loads and queries. `refreshAll({ includeLoadFunctions: false })` refreshes remote data without rerunning loads. The no-argument `refreshAll()` includes loads. [Navigation](https://svelte.dev/docs/kit/$app-navigation).
