---
name: restep-design
description: Build an HTTP JSON API, or change an existing one, so it follows Restep (REST level 2½). Use when the user mentions Restep, or wants an API's errors, pagination, filtering, retries or concurrency made consistent.
---

# Building a Restep API

Restep is a profile for HTTP JSON APIs at Richardson level 2. It is split into profiles a server adopts one at a time and announces: `core` (required), then `query`, `writes`, `relations` and `sync`, plus the extensions `actions`, `batch`, `filter-expr` and `query-method`.

The specification is `spec.md`, next to this file. It is the source of truth: when a rule below and the specification disagree, the specification wins. With network access, https://restep.wubytes.com/1/spec.md is the latest version; without it, the copy here is enough.

## Steps

1. **Map the API.** List every collection and item resource with its methods, and every query parameter it already takes. Done when each resource is on the list.
2. **Pick the profiles.** `core` always. Add a profile when the user asks for it or the API already does what it covers: filtering or paginating → `query`; retries, concurrent edits or caching → `writes`; nested or linked resources → `relations`; clients keeping a local copy → `sync`. Respect the dependencies (`sync` needs `query`). Done when the list is decided and you have read, in full, sections 3 to 5 of `spec.md` and the section of each chosen profile.
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
- **Fail loudly**: a reserved parameter the resource does not implement (`filter`, `sort`, `fields`, `include`, `count`, `limit`, `cursor`, `since`, any `$name`, in any bracket form) gets `400` with `code: "unsupported_parameter"` and `parameter` set to the name as received. On every method, including `POST`, unless an error that comes earlier in the order of section 5.4 applies: a missing item with an unsupported parameter is `404`.
- `POST` → `201` with `Location`; `DELETE` → `204`; an unsupported method (`PUT` included) → `405` with `Allow`; `HEAD` wherever `GET` works.
- `PATCH` takes `application/merge-patch+json`, and `application/json` with the same meaning, and returns the whole item. A body that is not a JSON object → `400 bad_request`; another content type → `415` with `Accept-Patch`. `null` deletes a field; on a field that cannot be empty, `null` → `422 invalid`, never a silent reset to a default.
- An item lives at its collection's path + `/` + its `id`. The server assigns `id`: a non-empty string of unreserved URI characters. A `POST` with `id`, or a `PATCH` changing it, → `422 invalid`. Writes ignore members starting with `_`.
- A number that needs exact decimal arithmetic (money, hours like `"1.5"`) or may exceed ±(2^53−1) is a string in every representation, declared as `decimal`. Instants are RFC 3339 with an offset.
- `GET` on the API root returns the discovery document (section 4.2); every response carries one `Link: <https://restep.wubytes.com/1/core>; rel="profile"` per profile and extension.

**query**
- Declare in discovery every filterable field with its type and operators, every sort key, and `max_limit`. Anything undeclared → `unsupported_parameter`; a declared one with a bad value → `invalid_parameter`.
- Compare by the declared type: decimals exactly, never through floating point. Missing values never match an operator other than `null`, and sort last in both directions. Break sort ties by `id`.
- `limit` outside 1..`max_limit` → `invalid_parameter`, never a silent clamp. Cursors are opaque and signed, bound to the collection, filters and sort that issued them (never to `limit`, `fields`, `include` or `count`); a tampered one, or one reused with another collection, filter or sort → `invalid_parameter` on `cursor`. `filter[f]` and `filter[f][eq]` are the same filter, so both together → `invalid_parameter`; `filter[price][ge]` with `filter[price][le]` is a range, not a repeat.

**writes**
- Every item has a strong `ETag`; a response with `fields` carries the full item's ETag in `_etag`. `If-Match` is checked atomically with the write: a stale one → `412 precondition_failed`. `If-None-Match` with the current one → `304` to `GET`, `412` to a write.
- Every non-idempotent `POST` accepts `Idempotency-Key` (a quoted Structured Fields string). The same key and the same JSON body replays the stored response, with `Idempotent-Replayed: true`; the same key with another body → `422 idempotency_mismatch`. Keep keys at least 24 hours and declare `idempotency.ttl`.
- Every response to `GET` or `HEAD`, errors included, carries `Cache-Control`.

## Schemas

`schema/discovery.json` and `schema/problem.json`, next to this file, are JSON Schemas for the discovery document and for error bodies. When the API has a test suite, validate both there.
