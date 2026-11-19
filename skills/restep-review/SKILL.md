---
name: restep-review
description: Review an API change, an endpoint or an OpenAPI document against Restep (REST level 2½). Use when the user asks whether an API follows Restep, or to review or audit one against it.
---

# Reviewing against Restep

The specification is `spec.md`, next to this file, and it is the only yardstick: every finding cites a section of it. The published copy at https://restep.wubytes.com/1/spec.md is newer if the two differ.

## Steps

1. **Find what the API announces.** Read its discovery document (`GET` on the API root, or the code that builds it) and its `Link: rel="profile"` headers. Done when you know its profiles and extensions. Without a discovery document, assume `core` only and report the missing document as a finding.
2. **Read the yardstick.** Read, in full, sections 3 to 5 of `spec.md` and the section of every announced profile and extension. Those are the requirements the API has taken on.
3. **Check the change.** For every endpoint the change touches, apply every MUST and MUST NOT of those sections, and the SHOULDs. Also apply the list of breaking changes in section 5.5: a change that removes, renames or tightens something without going through deprecation is a finding even if each endpoint conforms. Done when every touched endpoint has been checked against every requirement of the announced profiles.
4. **Run the checker** if a server is reachable: `node restep-check.mjs <api-root>` (read only) from next to this file. Add `--writes` only against a test environment, and only if the user agrees. Merge its failures into your findings.
5. **Report** each finding as: the section number, MUST or SHOULD, the endpoint or file and line, what the API does, and what the section requires. Most severe first: broken MUSTs, then breaking changes, then SHOULDs. Say which requirements could not be checked and why.
