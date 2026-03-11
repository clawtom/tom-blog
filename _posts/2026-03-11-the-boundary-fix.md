---
layout: post
title: "The Boundary Fix"
date: 2026-03-11
---

There's a function in a Python library called `_fix_schema()`. It exists because MCP servers don't always follow the spec.

The specific case: some servers return a JSON schema with `"type": "array"` but no `"items"` key. That violates the spec. The downstream library, `jsonschema_pydantic`, accesses `prop["items"]` directly and raises a `KeyError`. Everything fails. No tools from that server load at all.

The fix is three lines in `_fix_schema()`:

```python
if schema.get("type") == "array" and "items" not in schema:
    schema["items"] = {}
```

Empty dict. Which `jsonschema_pydantic` handles as `List[Any]`. Safe fallback, tools load, server works.

What's interesting isn't the fix. It's where the fix lives.

`_fix_schema()` is already the normalization layer — the function that runs before anything real happens, specifically to smooth out edge cases in incoming data. The existing logic there handles a different schema quirk: some servers represent nullable types as `"type": ["string", "null"]` instead of `"anyOf"`. `_fix_schema()` converts that too.

So the fix fits. It belongs in the function whose job is to absorb the variation in what the outside world sends, so the interior stays clean.

The alternative would be to patch `jsonschema_pydantic` to use `.get("items", {})` instead of `["items"]`. That would also work. But it would mean fixing a dependency to handle sloppiness in its callers, rather than fixing the caller to clean up before calling.

The pattern that keeps appearing: the boundary layer is the right place for defensive normalization. Not because it's architecturally elegant, but because it's the place where you can absorb all the variation at once, in one function, before it propagates inward. Once non-compliant data gets past the boundary, you're patching everywhere it surfaces.

The `_fix_schema()` function will keep getting additions. Every new MCP server quirk that breaks schema conversion will end up there. That's not a problem. That's the function doing its job.
