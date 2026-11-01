---
name: svelte-deep-knowledge
description: Use this skill whenever you are about to read, edit or create a Svelte file, or when you are working within a SvelteKit project. It should _always_ be used before starting the work, to clear wrong assumptions. This skill contains opinionated common guidance and pitfalls of modern Svelte.
---

# Svelte deep knowledge

Use this as an index of likely knowledge gaps. Verify the relevant APIs against the official Svelte MCP during planning and implementation.

Verified on 2026-11-01 against Svelte 5.57.3, SvelteKit 3.1.1, and SvelteKit 2.70.3.

## Start with the installed versions

1. Inspect the package versions and Vite configuration. Match guidance to the installed Kit major version and the enabled experimental flags.
2. Call the Svelte MCP's `list-sections`, then fetch the relevant sections together with `get-documentation`. The MCP and svelte.dev serve Kit 3 docs. For a Kit 2 project, read the matching page from the `version-2` branch of `sveltejs/kit` under `documentation/docs`.
3. For Kit 3 projects and Kit 2 → 3 migrations, read [the Kit 3 reference](references/sveltekit-3.md). For Kit 2 projects, read [the Kit 2 reference](references/sveltekit-research.md).
4. Run `svelte-autofixer` on new or edited Svelte components/modules and resolve applicable findings. Use the project's checks for integration and types.

For examples of new Svelte APIs, async rendering, attachments, and compiler tooling, read [the Svelte reference](references/svelte-runtime.md).

## Remote functions, boundaries, and await

For new SvelteKit apps adopting experimental async, prefer remote functions with direct `await` for component data and granular boundaries for loading/error UI. Choose svelte:boundary placement based on what must render on the server.

- Enable `experimental.remoteFunctions` and `compilerOptions.experimental.async` together. The official remote-functions setup requires both flags.
- `load`, form actions, and `+server` routes remain supported but not encouraged. Follow existing architecture unless the task includes migration.
- `.loading`, `.error`, and `.current` remain supported query APIs. Direct `await` is the preferred mechanism.
- Use `await query(args)` in rendering and imperative contexts.
- `{#await}` remains supported though not encouraged, and has its own promise-state behavior. Use direct await and boundaries for coordinated async rendering; preserve `{#await}` when its behavior fits the task.

Verify with MCP sections `kit/remote-functions`, `svelte/await-expressions`, and `svelte/svelte-boundary`.

### Async rendering behavior

- Component-level await arrived in Svelte 5.36; async SSR arrived in 5.39. Await is supported in the top-level script, `$derived`, and markup.
- Svelte coordinates dependent UI updates while async work resolves. Independent template await expressions run concurrently. Sequential script awaits still create waterfalls; start independent work before awaiting its results.
- A boundary's `pending` snippet appears on initial creation. Use `$effect.pending()` for subsequent work in that boundary; its count excludes child boundaries.
- During SSR, a boundary with `pending` renders the placeholder and skips its content. Await expressions outside such boundaries resolve before `await render(...)` completes. Place content needed in the initial HTML accordingly. This API does not yet provide streaming SSR.
- `$state.eager(value)`, added in 5.41, exposes immediate feedback while coordinated updates wait. Use it for user feedback such as an active navigation indicator.
- `tick()` waits for pending state changes; `settled()`, added in 5.36, also waits for async work caused by them and the resulting DOM updates.

### Error boundaries

- `<svelte:boundary>` supports `failed(error, reset)` and `onerror(error, reset)` for rendering/effect failures. Handle event-handler and detached async errors at their call sites.
- `failed` snippets also catch errors during SSR, since Svelte 5.53. In SvelteKit, the error goes through `handleError` first, so the snippet receives the sanitized `App.Error`. `onerror` runs when the failed boundary hydrates.
- Kit 3 sends rendering errors through `handleError` to the nearest `+error.svelte`, which acts as a boundary. An async client `handleError` needs `compilerOptions.experimental.async`. Kit 2.54+ needs `experimental.handleRenderingErrors` for the same integration.
- The boundary documentation's older “since 5.51” note conflicts with the published changelog. Use 5.53 as the introduction version and check current patches when diagnosing SSR/hydration issues.

## Template features

### Declaration tags, Svelte 5.56+

- Prefer `{let ...}` and `{const ...}` for markup-local variables. `{@const}` is documented as legacy and remains available for older projects.
- Plain declarations initialize values. Use explicit runes for reactivity: `{let count = $state(0)}` and `{const doubled = $derived(count * 2)}`.
- Migrating reactive `{@const area = width * height}` requires `{const area = $derived(width * height)}`. A plain `{const area = width * height}` is not reactive.
- Declarations use lexical scope within their element/block, can shadow outer variables, and should precede use. Multiple declarators are supported.

### Attachments

- Prefer `{@attach ...}` for new element integrations on Svelte 5.29+. Return cleanup from the attachment.
- Attachments track reactive reads and re-run when dependencies change. Falsy values disable them. Avoid repeating expensive setup by reading changing options through a getter inside a child effect.
- Component wrappers must forward attachment-bearing props to the intended element. `createAttachmentKey()` supports attachments in spread objects.
- `fromAction(action, () => argument)` preserves an existing action's behavior. Its second argument is a getter. This helper arrived in 5.32, after attachments themselves.

### Classes, bindings, and HTML

- `class` accepts nested arrays and objects since 5.16. `ClassValue` from `svelte/elements` is available since 5.19 for wrapper props.
- Function bindings use `bind:value={get, set}` since 5.9. Use `null` as the getter for readonly bindings.
- Numeric inputs may produce `undefined` when empty or invalid. Checkbox state can use `bind:checked` and `bind:indeterminate`; radio groups use `bind:group`.
- Inputs support `defaultValue`/`defaultChecked` since 5.6. Selects support `defaultValue` since 5.57, including form reset behavior. Check binding precedence when supplying defaults.
- Customizable native selects are supported since 5.47. Check browser support and the documented `<selectedcontent>` pattern before replacing a custom select widget.
- JavaScript-style comments between attributes are valid since 5.53: `<div /* explanation */ class="card">`.
- `{@html}` accepts `TrustedHTML` since 5.52. Trusted Types support does not sanitize arbitrary input; use a trusted source or sanitization policy.

### Each blocks and snippets

- `{#each items as { id, name }}` destructures items. Range iteration can use `{#each { length: 8 }, index}`.
- Prefer stable item keys for lists whose identity matters. When editing an item, bind its property through the item object so mutations reach the source.
- Snippets are declared in markup and rendered with `{@render snippet(...)}`. Top-level snippets can be exported from `<script module>` since 5.5 if they do not capture instance state.
- Snippets support generics since 5.30. Component generics use `<script lang="ts" generics="T">`.
- In runes mode, component variables render dynamically as `<Component />`. Recursive components can import themselves.

## Runes and shared state

### State and props

- `$state.raw` tracks reassignment without deep proxying. Use it for large values replaced as a whole. `$state.snapshot` extracts plain data and respects `toJSON()`.
- Class instances are not deeply proxied. Use `$state` fields, or their first constructor assignment since 5.31. Reactive fields become non-enumerable accessors; preserve `this` when passing methods as callbacks.
- Passing a reactive value to a function passes its current value. Pass a getter when the function needs future values. Destructuring a proxy also reads current values.
- Use `.svelte.js`/`.svelte.ts` for shared rune logic. Export a stable state object or accessors when consumers need changing state; directly exporting reassigned rune state is restricted.
- Scope user-specific state through component context during SSR. Shared module state can leak across requests.
- `$props.id()`, added in 5.20, gives a component-instance ID that matches between SSR and hydration.
- `$bindable()` declares a bindable prop. Props can be locally reassigned; mutating a parent's non-bindable proxy raises ownership warnings. Fallback values are not automatically proxied.
- Keep the actual uninitialized type of `$state<T>()`, which includes `undefined`, or provide a valid initial value. Use narrowing at access sites.

### Deriveds and effects

- Use `$derived(expression)` or `$derived.by(fn)` for computed values. Deriveds declared with `let` can be reassigned for optimistic UI since 5.25; dependency changes recompute them.
- Destructured `$derived` values stay reactive. Returned objects are not automatically deep proxies. Referential equality can skip downstream work.
- Effects track synchronous reads from their latest run. Reads inside an ordinary async callback after await are not automatically dependencies.
- `$effect` runs in the browser. Return cleanup synchronously; reserve effects for external synchronization and put user-triggered work in handlers.
- `$effect.tracking()` detects a tracking context. `$effect.root()` creates a manually managed scope and returns a disposer.
- `getAbortSignal()`, added in 5.35, must run inside an effect or derived. Its signal aborts on re-execution or destruction. Use it for fetch work owned by that reactive computation.
- `untrack(fn)` excludes reads from dependency tracking.
- `$inspect(...).with(fn)` receives `"init"`/`"update"` followed by values. `$inspect.trace(label?)`, added in 5.14, must be the first statement in the traced function.
- `$host()` is for custom elements and returns their host DOM element.

## Runtime modules

- `createContext<T>()` provides typed context since 5.40. Since 5.57 it returns `[get, set, has]`; `has()` checks availability before `get()`, which throws when unset. Context lookup includes the current component and ancestors. Call context accessors during initialization, and set context before the first `await` in the script. Since 5.57.1, a later `setContext` throws `set_context_after_init` during SSR.
- `svelte/reactivity` provides `SvelteMap`, `SvelteSet`, `SvelteDate`, `SvelteURL`, and `SvelteURLSearchParams`. Map/Set entries are not made deeply reactive.
- Svelte 5.57 adds `SvelteMap.getOrInsert(key, value)` and `getOrInsertComputed(key, key => value)`. The computed callback runs when the key is absent, including correct handling of existing `undefined` entries.
- `MediaQuery` and `createSubscriber` arrived in 5.7. Use `createSubscriber` to connect external events to tracked reads with shared subscription setup and cleanup. `MediaQuery` accepts an SSR fallback; use CSS when it can express the same behavior.
- `svelte/reactivity/window` provides reactive `.current` values for dimensions, scroll position, online status, screen position, and pixel ratio since 5.11. Account for unavailable window values during SSR.
- `on()` from `svelte/events` preserves ordering with declarative event handlers and returns cleanup. Use `<svelte:window>`/`<svelte:document>` for declarative global listeners.
- `flushSync(fn?)` flushes pending updates synchronously. `createRawSnippet` supports programmatic markup with setup/cleanup. `getAllContexts()` supplies the current context map for programmatic mounting; call it during initialization.
- `Spring` and `Tween` from `svelte/motion` replace deprecated `spring()`/`tweened()` stores since 5.8. Both expose `.target`, `.current`, `.set()`, and `.of(fn)`. `Spring.set` supports `preserveMomentum`; `prefersReducedMotion` supports accessible motion.
- Since 5.55, motion exports include `TweenOptions`, `SpringOptions`, `SpringUpdateOptions`, and `Updater`.
- `hydratable(key, fn)`, added in 5.44, serializes data for reuse during hydration. It uses devalue and supports nested promises. Namespaced keys avoid collisions. Kit remote functions already handle this layer.
- With CSP, custom SSR integrations must configure `render` with a nonce per response or static hashes and put matching values in the response headers. Svelte 5.46 added render CSP support; 5.57 exports `RenderOutput`, `SyncRenderOutput`, `Csp`, and `Sha256Source` from `svelte/server`.
- For tooling, `print()` converts a modern Svelte AST to source since 5.45; `parseCss()` arrived in 5.48. Compiler options `css`, `runes`, and `customElement` can be filename-dependent functions since 5.54. Read the Svelte reference before using these APIs.

## SvelteKit

New apps use Kit 3 (`npx sv create`). Migrate a Kit 2 app with `npx sv migrate sveltekit-3`, then work through the generated TODOs. Kit 2 guidance applies only to projects that stay on 2.x.

### Kit 3 baseline

- Kit 3 requires Node 22.17, TypeScript 6, Svelte 5.57.1, Vite 8.0.12, and vite-plugin-svelte 7.
- Configuration lives in `sveltekit({...})` inside `vite.config.ts`, with Kit options at the top level beside `compilerOptions`. Kit 3 no longer reads `svelte.config.js`.
- `#lib` replaces `$lib` through package.json subpath imports. Write file extensions, such as `#lib/server/db.ts`.
- A `server` filename segment, or any `server` directory outside `src/routes`, marks a module server-only. A `remote` filename segment, as in `data.remote.ts` or `remote.ts`, marks a remote module.
- Declare environment variables in `src/env.ts` with `defineEnvVars` from `@sveltejs/kit/env`. Import values from `$app/env/private` and `$app/env/public`, and `browser`, `building`, `dev`, `version` from `$app/env`. Public variables can appear in `app.html` as `%sveltekit.env.NAME%`. `$env/*` and `$app/environment` are deprecated aliases.
- `$app/stores` is removed; use `$app/state`. `page.url` is readonly; copy it with `new URL(page.url.href)` before editing.
- `$app/paths` exports `resolve`, `asset`, and `match`; `base`, `assets`, and `resolveRoute` are removed. Generated `Path` and `AssetPath` types omit the leading slash.
- Define param matchers in one `src/params.ts` with `defineParams` from `@sveltejs/kit/params`. A matcher is a function or a Standard Schema, and its output types the param.
- Remote function types live in `$app/server`, navigation types in `$app/navigation`, `Page`/`ReadonlyURL` in `$app/state`, `ActionResult`/`SubmitFunction` in `$app/forms`, and hook types in `@sveltejs/kit/hooks`. `RequestEvent` and `Cookies` stay in `@sveltejs/kit`.
- `json()` and `text()` are deprecated; use `Response.json(...)` and `new Response(...)`.

### Kit 3 navigation, errors, and deployment

- Shallow routing uses `goto(url, { shallow: true, state })`; `pushState`/`replaceState` are deprecated. Add `replace: true` to replace the entry and `persistState: true` to keep `page.state` after reload. `reset: false` replaces `noScroll` and `keepFocus`. Navigation hooks fire for shallow changes too; check `navigation.shallow`.
- `refreshAll()` takes no options and keeps `page.state`; `invalidateAll()` is deprecated. A click on a link to the current URL runs `refreshAll()`.
- `goto` rejects destinations outside the app; use `window.location.href` for those. `preloadData` can return `{ type: 'error' }`. `preloadCode` takes a route ID such as `/blog/[slug]`.
- Call `snapshot({ capture, restore, id, reset })` from `$app/navigation` during component initialization. It replaces `export const snapshot`.
- `handleError` receives every error with a `kind`: `'app'`, `'framework'`, `'validation'` (server only), or `'unknown'`. Return `status` to change the response status. It replaces `handleValidationError`. `App.Error` always includes `status`; pass extra fields as `error(status, message, details)`.
- `redirect` to another origin needs `{ external: true }` or an origin allowlist.
- Enable tracing with top-level `tracing: { server: true }`. `src/instrumentation.server.ts` loads automatically, after env vars are populated.
- Set `paths.origin` when request headers cannot give the public origin, for example behind a proxy. It replaces `prerender.origin` and adapter-node's `ORIGIN` variable.
- `version.pollInterval` defaults to one hour. `updated.current` also turns on after server data loads, remote calls, and tab focus.
- adapter-cloudflare 8 removes `platform`: import `env` and `waitUntil` from `cloudflare:workers`, read `request.cf`, and use the global `caches`. adapter-vercel drops the edge runtime. `@sveltejs/adapter-bun` serves the app with Bun 1.4+.

### Remote functions, experimental since 2.27

Define remote functions in files with a `remote` filename segment and import `query`, `form`, `command`, or `prerender` from `$app/server`. Validate untrusted input with a Standard Schema implementation and authorize access in the handler.

- `query` reads data; `query.batch` batches calls made in the same macrotask. Reuse awaitable query instances and verify cache/subscription lifetime for imperative consumers.
- In Kit 3, reading `event.url`, `event.params`, or `event.route` inside a query throws. Pass route values as arguments.
- `query.live`, added in 2.59, yields current data from an async generator. It exposes `.connected` and `.reconnect()` and supports `for await` since 2.61. Consumers share a connection and intermediate values can be coalesced. Use it for current state, not an event log. Live queries update themselves instead of using `.refresh()`.
- `form` supports progressive enhancement through spread attributes, typed `.fields`, field `.as(...)`, validation issues, `.preflight()`, `.validate()`, `.enhance()`, and `.for(id)` for independent instances. Fields starting with `_` are not repopulated after failure. Kit 3 rejects submissions from controls not created with `field.as(...)`.
- Since 2.56, `field.as(type, defaultValue)` supports defaults. In Kit 3, radio and checkbox options take the checked state third: `.as('checkbox', tag, tags.includes(tag))`. Since 2.57, enhanced `submit()` resolves to a validity boolean. Check it before clearing input or performing success-only UI actions.
- Since 2.69, `form.submitted` is a boolean indicating whether the form has been submitted at least once; Kit 3 clears it on reset. `form.result` becomes available after the handler returns; `form.pending` counts in-flight submissions.
- Kit 3 fields have `touched()` and `dirty()`. Kit 3 uses `validate({ all: true })` where Kit 2 uses `validate({ includeUntouched: true })`; the 3.1 docs page still shows the Kit 2 name, so trust the installed types.
- Since 2.68, `RemoteFormEnhanceInstance` and `RemoteFormEnhanceCallback` provide types for reusable enhancement wrappers.
- `command` performs imperative mutations. Invoke it from user interactions or other appropriate imperative code; calling commands during rendering is disallowed. Prefer a remote form for a natural HTML form workflow.
- `prerender` produces build-time data, supports enumerated `inputs` and `dynamic: true`, and can use the browser Cache API. Check its public-data and caching constraints.

### Refreshes and validation

- Single-flight updates combine mutation results and refreshed data in one response. Server handlers can use `query(args).refresh()` or `.set(data)`; queries can update other queries since 2.65.
- Client-requested `.updates(...)` require server cooperation. `requested(queryFunction, limit)` yields `{ arg, query }`, plus `ignore` in Kit 3; the bound query preserves the original cache key even if validation transforms `arg`. Supply the required limit and update those bound queries. `query.batch` is supported; live queries use reconnect APIs. `.withOverride(fn)` adds an optimistic value to a requested update.
- In Kit 3, the server must refresh, set, reconnect, or `ignore()` every requested update within the limit; otherwise that query errors on the client. `requested(...).refreshAll()` and `.ignoreAll()` handle the whole list.
- `invalid()` reports programmatic field issues. Invalid remote calls reach `handleError` with `kind: 'validation'` in Kit 3 and `handleValidationError` in Kit 2. `getRequestEvent()` exposes request context; follow its async-context constraints on the target runtime.
- Use the reference for the installed Kit version for full form, live-query, and refresh examples before implementing these experimental APIs.

### Routing, state, and hooks

- `$app/state`, added in 2.12, exposes rune-reactive `page`, `navigating`, and `updated` objects. `$:` does not track their updates. Use `page.state` for shallow-routing state.
- `resolve`, added in 2.26, resolves a pathname or a typed route ID plus params. `asset()` resolves static assets. `match()`, added in 2.52, resolves a pathname back to a route ID and params.
- Generated `$app/types` includes `RouteId` and `RouteParams`, plus `Pathname` in Kit 2 or `Path`, `PageRouteId`, and `EndpointRouteId` in Kit 3. Parameter matchers narrow generated parameter types since 2.55.
- `onNavigate` can await a view transition and return a callback for after the DOM update. Navigation targets expose scroll positions since 2.51. `preloadCode` loads route code without running load functions.
- `transport` is a universal hook for custom serialization; Kit 3 also uses it for `page.state` and snapshots.
- `csrf.trustedOrigins` controls cross-origin form submissions. `output.bundleStrategy` accepts `split`, `single`, and `inline`. `csp.mode` accepts `hash`, `nonce`, and `auto`. `prerender.handleInvalidUrl`, added in 2.67, handles malformed URLs found while crawling; `handleHttpError` handles HTTP failures separately.

### Kit 2 projects

Kit 2.70.3 is the final 2.x release. Read [the Kit 2 reference](references/sveltekit-research.md) before writing Kit 2 configuration, environment, or navigation code.

- Configuration lives under `kit` in `svelte.config.js`. Since 2.62 it can also be flattened into `sveltekit({...})`, which makes Kit ignore `svelte.config.js`. Follow the shape the project uses.
- Environment variables use `$env/static/*`, `$env/dynamic/*`, and `$app/environment`. `experimental.explicitEnvironmentVariables`, added in 2.63, opts into the Kit 3 model; since 2.70, `defineEnvVars` comes from `@sveltejs/kit/env`.
- Shallow routing uses `pushState`/`replaceState` with `preloadData`. Snapshots use `export const snapshot` with `capture`/`restore`, typed as `Snapshot<T>`.
- `goto` accepts `noScroll`, `keepFocus`, and an `invalidate` array of strings, URLs, or `(url: URL) => boolean` predicates. `refreshAll({ includeLoadFunctions })` refreshes queries and loads; pass `false` to skip loads. `invalidateAll()` reruns active-page loads and queries.
- `handleError` skips expected errors. Tracing uses `instrumentation.server.ts` with the experimental tracing and instrumentation flags, added in 2.31.

## TypeScript and component conventions

- Use `Component<Props>`, `ComponentProps<typeof Component>`, and `Snippet<[Arguments]>` for component APIs. Element wrappers can use `HTMLButtonAttributes` or `SvelteHTMLElements['div']` from `svelte/elements`.
- Augment `svelte/elements` for custom attributes/events. Native TypeScript support covers syntax that erases; features needing runtime transformation require preprocessing.
- In new runes components, use `$props`, callback props, event attributes, snippets, and `$derived`. Preserve legacy syntax when maintaining a legacy component unless migration is part of the task.
- Use CSS custom properties for parent-controlled child styling. Prefer `.svelte.ts` state classes and typed context for new shared reactive logic; retain store contracts where integrations require them.

## Maintaining this skill

Review the complete package changelogs, recent official monthly posts, and MCP examples. Check the published types when docs and changelog disagree. Follow later reversals of experimental changes before recording an API. Update the verified date and version scope, and keep detailed usage in the linked references.

Release sources: [Svelte changelog](https://github.com/sveltejs/svelte/blob/main/packages/svelte/CHANGELOG.md), [Kit 3 changelog](https://github.com/sveltejs/kit/blob/main/packages/kit/CHANGELOG.md), [Kit 2 changelog](https://github.com/sveltejs/kit/blob/version-2/packages/kit/CHANGELOG.md), and [official posts](https://svelte.dev/blog).
