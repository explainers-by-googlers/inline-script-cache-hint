# Explainer for `cache-hints` on Inline Scripts

This proposal introduces a `cache-hints` attribute for HTML `<script>` tags, specifically targeting inline scripts. It allows developers to provide hints to the browser about whether an inline script should be cached.

## Proponents

- [Takashi Nakayama](https://github.com/azaika) (tnak@chromium.org)

## Participate
- https://github.com/explainers-by-googlers/inline-script-cache-hint/issues

## Introduction

Inline scripts are widely used but traditionally do not benefit from the browser's script cache because they lack a unique URL. This means large inline scripts must be parsed and compiled every time a page loads, which can negatively impact loading performance.

Chromium is implementing a new caching layer called **Inline Script Cache** to address this. To make this caching more effective and give developers control, we propose the `cache-hints` attribute.

## Background

The compilation time of inline scripts can have a non-negligible impact on loading performance, especially on low-end devices or with large scripts. While external scripts benefit from HTTP caching and code caching keyed by URL, inline scripts are typically re-compiled on every execution.

To improve this, a dedicated caching layer for inline scripts is being developed in Chromium. For more technical details on the underlying mechanism, please refer to the [Inline Script Cache Design Doc](https://docs.google.com/document/d/1pVFb79e5vkKJI7nZ15BXZFbNji_mB2Y8eZPGhEpK3jE/edit?tab=t.0#heading=h.lsbdweaff7ii).

## Goals

- Allow developers to explicitly opt-in important or static inline scripts for caching.
- Allow developers to opt-out dynamic or one-off inline scripts to avoid cache pollution.

## Non-goals

- Changing the execution timing or synchronous nature of inline scripts.

## Proposed Solution

We propose adding a `cache-hints` attribute to the `<script>` tag. This attribute only applies to inline scripts and has no effect on resource scripts (scripts with a `src` attribute).

### Attribute Values

- `eager`: Signals to the browser that this script is a good candidate for caching (e.g., it's large, static, and used across pages). The specific caching behavior and strategy are left to the browser's discretion.
- `never`: Signals that the browser **must not** cache this script. In contrast to `eager`, this is a strict instruction to prevent caching of dynamic or one-off scripts.

### WebIDL

```idl
[Exposed=Window]
interface HTMLScriptElement : HTMLElement {
  [HTMLConstructor] constructor();

  [CEReactions, Reflect] attribute DOMString type;
  [CEReactions, ReflectURL] attribute USVString src;
  [CEReactions, Reflect] attribute boolean noModule;
  [CEReactions] attribute boolean async;
  [CEReactions, Reflect] attribute boolean defer;
  [SameObject, PutForwards=value, Reflect] readonly attribute DOMTokenList blocking;
  [CEReactions] attribute DOMString? crossOrigin;
  [CEReactions] attribute DOMString referrerPolicy;
  [CEReactions, Reflect] attribute DOMString integrity;
  [CEReactions] attribute DOMString fetchPriority;
+ [CEReactions, Reflect] attribute DOMString cacheHints;  // New attribute

  [CEReactions] attribute DOMString text;

  static boolean supports(DOMString type);

  // also has obsolete members
};
```

### Examples

```html
<!-- Cache large, static, cross-page scripts -->
<script cache-hints="eager">
  function commonLargeFunction() {
    /* hundreds of lines */
  }
</script>

<!-- Don't cache scripts with dynamic data -->
<script cache-hints="never">
  var data_url = "data:image/...";
  insertImage(data_url);
</script>

<!-- These attributes have no effect on resource scripts -->
<script cache-hints="eager" src="https://example.com/script.js"></script>
<script cache-hints="never" src="https://example.com/script.js"></script>
```

## Alternatives Considered

### Automatic Detection by Browser

An alternative is for the browser to automatically determine which inline scripts to cache (e.g., based on size, execution frequency, or script complexity) without developer hints.

However, the HTML specification guarantees that inline script compilation and execution are synchronous. This strict timing requirement makes it difficult for the browser to run advanced, potentially time-consuming analysis algorithms to decide on caching before execution without regressing performance. Developer hints via `cache-hints` provide a low-overhead way to guide the browser.

## Security and Privacy Considerations

This feature does not expose new cross-origin data, as it only applies to scripts already embedded in the document. The cache keys will be properly partitioned to prevent cross-site tracking, consistent with standard browser caching policies.
