---
name: restep-design
description: Build an HTTP JSON API, or change an existing one, so it follows Restep (REST level 2½). Use when the user mentions Restep, or wants an API's errors, pagination, filtering, retries or concurrency made consistent.
---

# Building a Restep API

Restep is a profile for HTTP JSON APIs at Richardson level 2. It is split into six profiles a server adopts one at a time and announces: `core` (required), then `query`, `writes`, `cache`, `relations` and `sync`, plus the extensions `actions`, `batch`, `filter-expr`, `query-method`, `oauth` and `inherit`. `sync`, `batch`, `filter-expr`, `query-method` and `inherit` are experimental (section 1.2): they may change incompatibly in a minor version. A server that implements this draft declares `"restep": "1.0-draft.6"` (section 1.1).

The specification is `spec.md`, next to this file. It is the source of truth: when a rule below and the specification disagree, the specification wins. With network access, https://restep.wubytes.com/1/spec.md is the latest version; without it, the copy here is enough.

## Steps

1. **Map the API.** List every collection and item resource with its methods, and every query parameter it already takes. Done when each resource is on the list.
2. **Pick the profiles.** `core` always. Add a profile when the user asks for it or the API already does what it covers: filtering or paginating → `query`; retries or concurrent edits → `writes`; `Cache-Control`, `ETag` and `304` on reads → `cache`; nested or linked resources → `relations`; clients keeping a local copy → `sync`; values an item inherits from another → the `inherit` extension. Respect the dependencies (`sync` needs `query`), and tell the user when a choice is experimental. Done when the list is decided and you have read, in full, sections 3 to 5 of `spec.md` and the section of each chosen profile.
3. **Build `core`**, then each chosen profile, from its section. The rules below are the ones most often missed; they are a checklist, not a summary.
4. **Verify** with `restep-check.mjs`, next to this file, against a local server with test data:
   ```bash
   node restep-check.mjs http://localhost:3000/ --writes
   ```
   It finds the collections in the discovery document and creates items by copying one it reads; if that copy fails validation, pass a valid body with `--fixture <collection>=<json or @file>`. Fix every `FAIL`. Fix each `WARN`, or tell the user why it stays. Done when it exits 0 and you have told the user which `WARN` and `SKIP` results remain and why.

## Rules most often missed

**core**
- A collection is `{"items": [...], "next": null}`. `next` is always present, even before the API paginates.
- Every 4xx and 5xx body is `application/problem+json` with `type`, `title`, `status` (equal to the response's) and `code`. `code` comes from the table in section 5.4 for that status (`not_found`, `invalid`, `conflict`…); for a status the table lacks, the IANA name in snake_case.
- **Fail loudly**: a reserved parameter the resource does not implement (`filter`, `sort`, `fields`, `include`, `count`, `limit`, `cursor`, `since`, `languages`, any `$name`, in any bracket form) gets `400` with `code: "unsupported_parameter"` and `parameter` set to the name as received. On every method, including `POST`, unless an error that comes earlier in the order of section 5.4 applies: a missing item with an unsupported parameter is `404`.
- `POST` → `201` with `Location`; `DELETE` → `204`; an unsupported method (`PUT` included) → `405` with `Allow`; `HEAD` wherever `GET` works.
- `PATCH` takes `application/merge-patch+json`, and `application/json` with the same meaning, and returns the whole item. A body that is not a JSON object → `400 bad_request`; another content type, or none → `415` with `Accept-Patch`. A `POST` body that is not `application/json` (or has no `Content-Type`) → `415` too, and every `2xx` with a JSON body says `application/json` (section 3). `null` deletes a field; on a field that cannot be empty, `null` → `422 invalid`, never a silent reset to a default.
- An item lives at its collection's path + `/` + its `id`, a non-empty string of unreserved URI characters that never changes. Who chooses it is declared per collection in `id_from` (section 5.1): missing or `"server"`, a `POST` with `id` → `422 invalid` with a `read_only` entry at `/id`; `"client"`, a `POST` without `id` → `422` with `required`; `"either"`, both work. A client-chosen `id` that the collection has or has had → `409 conflict` with a `taken` entry; require a format that is hard to guess, such as a UUID. A `PATCH` changing `id` → `422 invalid` / `read_only`. Writes ignore members starting with `_`.
- A resource that is one of a kind (the shop's settings, the current user) is a singleton (section 5.9): declared in `singletons`, read with `GET` and, if it allows, changed with `PATCH` (`POST` and `DELETE` → `405`), with no collection around it, at a path no item could have.
- A response that depends on credentials (`Authorization`, cookies, a client certificate), `401`, `403` and a concealing `404` included, carries `Cache-Control: private` or `no-store` on every server, whether or not it announces `cache` (section 5.1).
- A number that needs exact decimal arithmetic (money, hours like `"1.5"`) or may exceed ±(2^53−1) is a string in every representation, declared as `decimal`. Instants are RFC 3339 with an offset.
- Data never depends on `Accept-Language` (section 3); only error texts, discovery descriptions and schemas may (section 5.6), with `Vary: Accept-Language` on every such response, the default language's too, never a `406`. Text in several languages goes in a language map, `{"en": "…", "es-mx": "…"}`, with lowercase tags, `{}` when empty, declared in `language_maps`.
- A value an item can inherit (a tax rate from its category, its store or a default) is the experimental `inherit` extension, listed in `extensions` (section 11.6). It is two fields: a read-only effective field (`tax_rate`) with the resolved value, and an own field (`tax_rate_own`) that is written, `null` meaning it inherits. Declare them in `inherit` with `own`, the to-one relations in `from` (only the next step) and `"default": true`. Writing the effective field with its current value is ignored, with another value gives `422 invalid` / `read_only`, so sending back what was read never pins an inherited value. Explain the source in `_inherited` (`{"from": {"href": …}}`, `"default"` or `"hidden"`); never a link to something the client cannot see.
- `GET` on the API root returns the discovery document (section 4.2); every response carries one `Link: <https://w3id.org/restep/1/>; rel="profile"`, to the version URI and no other (section 4.1); which profiles and extensions the API implements is said only by the discovery document.
- Errors carry a `detail` that says how to fix the request; `invalid_parameter` and `unsupported_parameter` name the parameter in `parameter`.
- So that agents and other generic clients can use the API without trying it (section 4.3), declare per collection `methods` and `item_methods` (they must match the `Allow` of every `405`), a one-sentence `description`, and `schemas` with JSON Schemas for `create` and `update` (a body they reject → `422 invalid`). Optional, but worth it whenever an agent may call the API.
- Write `create` so that a client can build a form from it (section 4.3, appendix A): properties in the order they should appear, a `title` (the label) and a `description` (the help) on each, translated by `Accept-Language`; choices as a `oneOf` of `const` values with their `title`; `examples` for placeholders. A `default` in a top-level property of `create` binds the server: a `POST` that leaves the field out stores that value (a `null` is not leaving it out), so leave out of `required` a field that has one. Never put a `default` in `update`. A starting value that depends on the user or the moment (today's date, their shop) cannot go in discovery: the server fills it in when the `POST` leaves it out.

**query**
- Declare in discovery every filterable field with its type and operators, every sort key, and `max_limit`. Anything undeclared → `unsupported_parameter`; a declared one with a bad value → `invalid_parameter`.
- A criterion that does not compare a field (an ORM scope such as "active", a text search, "approved by someone named…") is a named filter, declared in the collection's `named_filters` with a `description` of which items it returns and, if it takes a value, its `type`. The client sends `filter[name]` with no operator: no value for a switch, a value of its type otherwise. An operator on it → `unsupported_parameter`; a switch with a value, or a typed one empty or mistyped → `invalid_parameter`. Never name one like a key of `filter`, and never treat an empty value as "no filter".
- Compare by the declared type: decimals exactly, never through floating point. Missing values never match an operator other than `null`, and sort last in both directions. Break sort ties by `id`.
- `limit` outside 1..`max_limit` → `invalid_parameter`, never a silent clamp. Cursors are opaque and signed, bound to the collection, filters and sort that issued them (never to `limit`, `fields`, `include` or `count`); a tampered one, or one reused with another collection, filter or sort → `invalid_parameter` on `cursor`. `filter[f]` and `filter[f][eq]` are the same filter, so both together → `invalid_parameter`; `filter[price][ge]` with `filter[price][le]` is a range, not a repeat.

**writes**
- Every item has a strong `ETag`; every response whose body is one item carries the full item's ETag in `_etag`, with or without `fields`. `If-Match` is checked atomically with the write: a stale one → `412 precondition_failed`. `If-None-Match` with the current one → `412` to a write. A write to an item that no longer exists → `404`, before any precondition.
- Every non-idempotent `POST` accepts `Idempotency-Key` (a quoted Structured Fields string). The same key and the same JSON body replays the stored response, with `Idempotent-Replayed: true`; the same key with another body → `422 idempotency_mismatch`. Keep keys at least 24 hours and declare `idempotency.ttl`. The query counts towards "the same request" (section 7.1).

**cache**
- Every response to `GET` or `HEAD`, errors included, carries `Cache-Control`. `writes` no longer brings this: announce `cache` for it (section 8).
- A full item or singleton carries an `ETag` (the item ETag if the server also announces `writes`); `If-None-Match` that matches → `304` to `GET`, `HEAD` or `QUERY`.
- `Cache-Groups` on reads and `Cache-Group-Invalidation` on writes that committed changes (RFC 9875) are recommended; group names never reveal items the client cannot see.

## Schemas

`schema/discovery.json` and `schema/problem.json`, next to this file, are JSON Schemas for the discovery document and for error bodies. When the API has a test suite, validate both there.
