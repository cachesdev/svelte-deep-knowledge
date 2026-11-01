# Svelte runtime and syntax reference

Verified 2026-11-01 against Svelte 5.57.3, released October 26, 2026. The complete package changelog was scanned from `5.0.0-next.1` through 5.57.3, including experimental API removals and later fixes. Releases 5.57.1 through 5.57.3 contain fixes and parser performance work only. This file records additions and usage details relevant to current Svelte 5 work. [Release](https://github.com/sveltejs/svelte/releases/tag/svelte@5.57.0), [changelog](https://github.com/sveltejs/svelte/blob/main/packages/svelte/CHANGELOG.md).

## Declaration tags and tool compatibility

Svelte 5.56 supports lexical declarations throughout markup. A plain declaration initializes a value when its scope is created. Use `$derived` for values that should follow later dependency changes. [Declaration tags](https://svelte.dev/docs/svelte/declaration-tags).

```svelte
<script>
  let price = $state(10);
</script>

{const total = $derived((price ?? 0) * 1.1)}
<label>Price <input type="number" bind:value={price} /></label>
<p>{total.toFixed(2)}</p>
```

An existing `{@const total = price * 1.1}` already tracks dependencies. Preserve that behavior with an explicit `$derived` when migrating. Keep declarations before use and check scope when moving them between an element, each block, or snippet. The compiler received several follow-up fixes for declaration tags and async dependencies in 5.56.x and 5.57. [Declaration release](https://github.com/sveltejs/svelte/releases/tag/svelte@5.56.0).

The July post records declaration-tag support across language tools and formatting. A compiler upgrade alone does not establish editor, formatter, or lint-parser compatibility. Check the installed tools when diagnostics disagree. [July 2026](https://svelte.dev/blog/whats-new-in-svelte-july-2026).

## Context and reactive maps, Svelte 5.57

`createContext` now returns `[get, set, has]`. Create the context definition once in a shared module, then call its accessors during component initialization. `has()` checks for a provider without invoking the throwing getter. A provider in the current component is visible after it is set. [Context API](https://svelte.dev/docs/svelte/svelte#createContext).

```js
// theme-context.js
import { createContext } from 'svelte';

export const [getTheme, setTheme, hasTheme] = createContext();
```

```svelte
<script>
  import { getTheme, hasTheme } from './theme-context.js';

  const theme = hasTheme() ? getTheme() : { color: 'black' };
</script>

<p style:color={theme.color}>Account settings</p>
```

Set context before the first `await` in a component script. Since 5.57.1, a `setContext` call after an `await` throws `set_context_after_init` during SSR.

Keep user-specific context values inside each rendered app instance. The context definition can be shared; a mutable singleton containing request data must not be shared between SSR requests. Svelte 5.50 also added support for using typed contexts with programmatic component instantiation. Check the current imperative API and source when supplying context to `mount` or `render`. [Context](https://svelte.dev/docs/svelte/context), [March 2026](https://svelte.dev/blog/whats-new-in-svelte-march-2026).

`SvelteMap` has native-style read-or-initialize methods. They test whether the key exists, so a stored `undefined` is still an existing entry. Values inserted into the map are not automatically deep proxies. [Reactive map API](https://svelte.dev/docs/svelte/svelte-reactivity#SvelteMap).

```js
import { SvelteMap } from 'svelte/reactivity';

const labels = new SvelteMap();
labels.getOrInsert('draft', 'Draft');
labels.getOrInsertComputed('review', (key) => key.toUpperCase());
```

Read map values from markup, a derived, or an effect to subscribe to changes. Choose the computed method when constructing the default value is expensive. [September 2026](https://svelte.dev/blog/whats-new-in-svelte-september-2026).

## Select defaults, Svelte 5.57

A select can declare its default/reset value directly. An initial non-nullish binding value takes precedence; an uninitialized binding can adopt the default. [Select bindings](https://svelte.dev/docs/svelte/bind#select-bind:value).

```svelte
<script>
  let status = $state();
</script>

<form>
  <label>
    Status
    <select bind:value={status} defaultValue="draft">
      <option value="draft">Draft</option>
      <option value="published">Published</option>
    </select>
  </label>
  <button type="reset">Reset</button>
</form>
```

Customizable native select support arrived in 5.47, including `<selectedcontent>`. Browser support and Svelte compiler support are separate checks. [Select implementation](https://github.com/sveltejs/svelte/pull/17429).

## Async rendering and boundary placement

With `compilerOptions.experimental.async`, independent await expressions in markup can run concurrently. Sequential awaits inside a script retain JavaScript sequencing. To avoid a startup waterfall, create independent promises first, then await their results. [Await expressions](https://svelte.dev/docs/svelte/await-expressions).

A boundary with a `pending` snippet renders that snippet on the server and skips the main content. This is suitable for optional content loaded after hydration. To include resolved content in the initial HTML, let async SSR await it outside a boundary with a pending snippet. A boundary used for `failed` handling can omit `pending`. [Async SSR](https://svelte.dev/docs/svelte/await-expressions#Server-side-rendering).

`$effect.pending()` counts pending work in the current boundary and excludes nested boundaries. Its scope matters: place the indicator inside the boundary whose work it should report. `pending` is the first-render placeholder; later coordinated updates use the existing content plus any explicit progress indicator. [Pending work](https://svelte.dev/docs/svelte/$effect#effect.pending).

`fork(fn)` starts speculative work based on synchronous state changes. Use `await fork.commit()` to apply it, or `fork.discard()` when it will not be used. This is useful for preloading interactive content on pointer/focus intent; ordinary navigation can rely on Kit's preloading. [Fork API and example](https://svelte.dev/docs/svelte/await-expressions#Forking).

### Error handling and hydration

Server-side boundaries require `transformError` when using `render` directly. The transformed value is serialized into the page and used by `failed`; keep it safe to expose. The client `onerror` callback runs when that failure is hydrated. Event handlers and detached asynchronous tasks need their own error handling. [Boundaries](https://svelte.dev/docs/svelte/svelte-boundary).

The published changelog and March post put SSR boundary support in 5.53.0. A boundary documentation note says 5.51; use the published release evidence for the version gate. Subsequent fixes include escaping transformed errors in 5.53.5, reset/onerror hydration in 5.56.8, and correct failed-snippet scope in 5.57. [March post](https://svelte.dev/blog/whats-new-in-svelte-march-2026), [changelog](https://github.com/sveltejs/svelte/blob/main/packages/svelte/CHANGELOG.md).

### Hydratable data and CSP

`hydratable(key, fn)` evaluates and serializes data on the server, reuses it during hydration, and calls `fn` normally afterward. It supports devalue-compatible values and nested promises. Use a key namespace in libraries. This API does not make server-only code safe to import into a browser component; keep database access behind the framework's server boundary. [Hydratable data](https://svelte.dev/docs/svelte/hydratable).

For custom SSR with CSP, pass `csp: { nonce }` to `render` and use the same per-response nonce in the header. For generated static HTML, use `csp: { hash: true }` and include the returned `hashes.script` values in the policy. Render output uses `body` and `head`. In 5.57, renderer integrations can import the public `RenderOutput`, `SyncRenderOutput`, `Csp`, and `Sha256Source` types. [Server API](https://svelte.dev/docs/svelte/svelte-server).

## Attachments without repeated setup

Attachments run in an effect and can return cleanup. Changes to `{@attach factory(options)}` can recreate the attachment. If setup is expensive, accept a getter and read it inside a child effect so option changes update the existing integration. For example, `createChart` below is an application/library function returning `setOptions` and `destroy` methods. [Attachment lifecycle](https://svelte.dev/docs/svelte/@attach#Controlling-when-attachments-re-run).

```js
// chart.svelte.js
import { createChart } from './chart-library.js';

export function chart(getOptions) {
  return (element) => {
    const instance = createChart(element);
    $effect(() => instance.setOptions(getOptions()));
    return () => instance.destroy();
  };
}
```

Use it as `{@attach chart(() => options)}`. To adapt an existing action, `fromAction(action, () => argument)` preserves the action's update behavior. The converter arrived in 5.32. [Attachment helpers](https://svelte.dev/docs/svelte/svelte-attachments).

## Compiler, snippets, and custom elements

- `print(parse(source, { modern: true }))` returns `{ code, map }`. `print`, added in 5.45, consumes the modern Svelte AST. Formatting can change. `parseCss`, added in 5.48, returns a CSS AST. These are useful for codemods and tooling. [Compiler API](https://svelte.dev/docs/svelte/svelte-compiler).
- Since 5.54, `css`, `runes`, and `customElement` compiler options may be functions receiving `{ filename }`. Scope forced runes compilation so that legacy dependency components can still compile. [Compiler options](https://svelte.dev/docs/svelte/svelte-compiler#CompileOptions), [April 2026](https://svelte.dev/blog/whats-new-in-svelte-april-2026).
- Since 5.49, custom elements accept a `ShadowRootInit` object in `shadow`, for settings such as `mode` and `delegatesFocus`. Native custom-element slots retain browser semantics. [Custom elements](https://svelte.dev/docs/svelte/custom-elements).
- Generic snippets use syntax such as `{#snippet row<T>(item: T)}` in a TypeScript component. Reusable exported snippets must not capture instance state. [Generic snippet change](https://github.com/sveltejs/svelte/pull/15915), [snippet exports](https://svelte.dev/docs/svelte/snippet#Exporting-snippets).
- Motion types `TweenOptions`, `SpringOptions`, `SpringUpdateOptions`, and `Updater` became public in 5.55. Use the library's exported types for wrappers. [May 2026](https://svelte.dev/blog/whats-new-in-svelte-may-2026).

## Release maintenance

The full history contains superseded experimental names such as `$state.frozen`, `$state.link`, and `$effect.active`. Follow the current docs and the later removal/rename entries before adopting anything found in an old release. Patch fixes also matter for async coordination, hydration, declaration tags, and serialization; avoid preserving obsolete application workarounds after an upgrade. [Complete changelog](https://github.com/sveltejs/svelte/blob/main/packages/svelte/CHANGELOG.md).

Official posts reviewed for feature context span March through October 2026. For future updates, use the [monthly posts](https://svelte.dev/blog) to locate examples and then verify their claims against current MCP docs and published releases.
