# Explainer for `cache-hint` on Inline Scripts

This proposal introduces a `cache-hint` attribute for HTML `<script>` tags, specifically targeting inline scripts. It allows developers to provide hints to the browser about whether an inline script should be cached.

## Proponents

- [Takashi Nakayama](https://github.com/azaika) (tnak@chromium.org)

## Participate
- https://github.com/explainers-by-googlers/inline-script-cache-hint/issues

## Introduction

Inline scripts are widely used but traditionally less benefit from the browser's script cache than caching for resource scripts. This means large inline scripts must be parsed and compiled every time a page loads, which can negatively impact loading performance.

Chromium is implementing a new caching layer called **Inline Script Cache** to address this. To make this caching more effective and give developers control, we propose the `cache-hint` attribute.

## Background

The compilation time of inline scripts can have a non-negligible impact on loading performance, especially on low-end devices or with large scripts. While external scripts benefit from HTTP caching and code caching keyed by URL, inline scripts are typically re-compiled on every execution.

While browsers can try to automatically detect cacheable inline scripts based on heuristics, the HTML specification guarantees that inline script compilation and execution are **synchronous**. This strict timing requirement makes it difficult for browsers to run advanced, potentially slow analysis to decide on caching without risking performance regressions during the critical loading path. Furthermore, implementation experience in Chromium highlighted that performing cache lookups synchronously can itself become a bottleneck if applied to every inline script unconditionally.

Developer hints via `cache-hint` provide a low-overhead way to guide browsers effectively, helping to avoid expensive lookups or unnecessary compilation. For details on the underlying mechanism being explored, see the [Inline Script Cache Design Doc](https://docs.google.com/document/d/1pVFb79e5vkKJI7nZ15BXZFbNji_mB2Y8eZPGhEpK3jE/edit?tab=t.0#heading=h.lsbdweaff7ii).

## Goals

- Allow developers to explicitly opt-in important or static inline scripts for caching.
- Allow developers to opt-out dynamic or one-off inline scripts to avoid cache pollution.

## Non-goals

- Changing the execution timing or synchronous nature of inline scripts.

## Proposed Solution

We propose adding a `cache-hint` attribute to the `<script>` tag. This attribute only applies to inline scripts and has no effect on resource scripts (scripts with a `src` attribute).

### Attribute Values

- `eager`: Signals to the browser that this script is a good candidate for caching (e.g., it's large, static, and used across pages). The specific caching behavior and strategy are left to the browser's discretion.
- `default`: Signals that the browser should use its default caching strategy for inline scripts: automatically determines which inline scripts to cache.
- `never`: Signals that the browser **must not** cache this script. This is a strict instruction to prevent caching of dynamic or one-off scripts.

### WebIDL

`HTMLScriptElement` is extended with `cacheHints` attribute.

```idl
partial interface HTMLScriptElement : HTMLElement {
  [CEReactions, Reflect] attribute DOMString cacheHints;
};
```

### Examples

```html
<!-- Prefer caching large, static, cross-page scripts -->
<script cache-hint="eager">
  function commonLargeFunction() {
    /* hundreds of lines */
  }
</script>

<!-- Same as no attribute: browsers determine caching automatically -->
<script cache-hint="default">
  function anotherFunction() {
    /* some lines */
  }
</script>

<!-- Don't cache scripts with dynamic data -->
<script cache-hint="never">
  var data_url = "data:image/...";
  insertImage(data_url);
</script>

<!-- These attributes have no effect on resource scripts -->
<script cache-hint="eager" src="https://example.com/script.js"></script>
<script cache-hint="never" src="https://example.com/script.js"></script>
```

## Security and Privacy Considerations

Browsers must ensure that caching of inline scripts does not expose cross-origin data based on the embedding document's same-origin policy. Detailed cache management strategies are up to individual browser vendors.

## FAQ

### Why `cache-hint` is Not Applicable for Resource Scripts?

Caches for resource scripts are already well-supported via HTTP caches and other existing mechanisms. Adding cache hints for resource scripts would add unnecessary complexity to the platform.

### How Wide is the Cache Scope?

It depends on browser implementation, including whether cross-document within the same-origin is supported. However, browsers must not cache inline scripts cross-origins. As an example, the current implementation in Chromium isolates cache per top-level site and frame origin, same as HTTP cache partitioning.