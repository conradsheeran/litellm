# LiteLLM v1.98.0 MCP Routing UTF-8 Hotfix and Database Image Publication

**Status:** Proposed implementation specification
**Baseline:** upstream tag `v1.98.0` at commit `d8f71d7bdbd7c9873d98293f83d64c6db72847e6`
**Target fork:** `conradsheeran/litellm`
**Target image:** `ghcr.io/conradsheeran/litellm-database:v1.98.0`
**Scope:** source patch, regression coverage, and fork-specific image publication CI
**Out of scope for this document:** implementing the patch or changing CI

## 1. Context

The production request path must remain:

```text
MCP Client
  -> litellm NPM client
  -> LiteLLM MCP Gateway
  -> mcp.conraaad.com NPM client
  -> MCP Server
```

A valid Streamable HTTP JSON-RPC tool call can exceed 4096 bytes and contain multibyte UTF-8 text. LiteLLM v1.98.0 reads at most `_MCP_ROUTING_PEEK_MAX_BYTES` bytes from the request body to decide whether the request is an MCP `initialize` call and whether a stateful JSON-RPC response should bypass the session lock.

The peek is a raw byte prefix. If byte 4096 cuts a multibyte UTF-8 code point, `json.loads(peeked_bytes)` raises `UnicodeDecodeError`. The two routing parse sites currently catch only `json.JSONDecodeError` and `TypeError`, so `UnicodeDecodeError` escapes and LiteLLM returns HTTP 500 before the complete body reaches normal downstream request handling.

Representative production failure:

```text
'utf-8' codec can't decode bytes in position 4094-4095: unexpected end of data
```

This is a routing-peek failure inside LiteLLM. It is not caused by Nginx buffering, MCP Server body parsing, or corruption of the complete request body.

## 2. Goals

1. Prevent a valid large UTF-8 MCP request from failing when the 4096-byte routing peek ends inside a code point.
2. Preserve the existing bounded-memory routing behavior and byte-for-byte downstream replay.
3. Base the forked build exactly on LiteLLM `v1.98.0` without unrelated version upgrades.
4. Publish only the database-enabled image under the fork owner:
   `ghcr.io/conradsheeran/litellm-database:v1.98.0`.
5. Make the build reproducible, auditable, and reversible.

## 3. Non-goals

- Do not bypass LiteLLM Gateway.
- Do not change `_MCP_ROUTING_PEEK_MAX_BYTES`.
- Do not buffer the complete request body for routing.
- Do not change Nginx or downstream MCP Server middleware.
- Do not modify MCP authentication, session ownership, lock semantics, tool routing, or request schemas.
- Do not upgrade LiteLLM beyond `v1.98.0` as part of this hotfix.
- Do not publish `litellm`, `litellm-non_root`, `litellm-slim`, Helm charts, Python packages, or any image other than `litellm-database`.
- Do not publish `latest`, `main`, `main-stable`, or another moving tag.
- Do not modify the inherited upstream Git tag `v1.98.0`.

## 4. Baseline and release identity

### 4.1 Immutable upstream baseline

Implementation must start from:

```text
Tag:    v1.98.0
Commit: d8f71d7bdbd7c9873d98293f83d64c6db72847e6
```

Before editing, verify:

```bash
git rev-parse v1.98.0
git merge-base --is-ancestor v1.98.0 HEAD
```

The inherited `v1.98.0` Git tag must continue to identify the unmodified upstream source. It must not be deleted, moved, or force-pushed.

### 4.2 Fork patch release tag

After the patch, tests, and publication workflow are complete, create an annotated fork-only source tag:

```text
v1.98.0-mcp-utf8.1
```

That tag identifies the exact patched source revision and triggers publication. It is distinct from the upstream baseline tag, so source provenance remains unambiguous.

### 4.3 Published image identity

The workflow publishes exactly:

```text
ghcr.io/conradsheeran/litellm-database:v1.98.0
```

This deliberately preserves the upstream repository and version naming while replacing only the owner `berriai` with `conradsheeran`. The image must carry OCI source and revision labels pointing to the fork and the patched commit.

The workflow must fail rather than overwrite this tag if it already exists. A later hotfix revision must receive a new explicit image tag instead of silently mutating the first published artifact.

## 5. Source patch

### 5.1 Files

Modify only the runtime owner of this behavior:

```text
litellm/proxy/_experimental/mcp_server/server.py
```

Add regression coverage to the existing mapped test file:

```text
tests/test_litellm/proxy/_experimental/mcp_server/test_mcp_server.py
```

### 5.2 Required code change

In `server.py`, replace both occurrences of:

```python
except (json.JSONDecodeError, TypeError):
```

with:

```python
except (json.JSONDecodeError, UnicodeDecodeError, TypeError):
```

The two required sites are:

1. `_is_initialize_request()`, where a truncated routing peek is parsed to detect the `initialize` method.
2. `handle_streamable_http_mcp()`, where the same peek is parsed to detect a JSON-RPC response that must bypass the stateful session lock.

After the patch, the file invariant must be:

```text
old pattern count:     0
patched pattern count: 2
```

### 5.3 Intended behavior

`UnicodeDecodeError` from the incomplete routing prefix is treated the same as the already-supported incomplete-JSON case:

- `_is_initialize_request()` returns `False`, so a no-session non-initialize call routes to the stateless manager.
- JSON-RPC response detection falls back to the existing replacement-decoded, depth-aware top-level-key scan.
- Consumed ASGI messages are replayed unchanged.
- Any unread body messages remain streamed from the original `receive` callable.
- The complete valid UTF-8 request reaches the selected MCP session manager byte for byte.

### 5.4 Deliberately excluded alternative

Do not add a new `_utf8_boundary_prefix()` helper in this fork hotfix. Upstream PR #34919 proposes both boundary trimming and defensive exception handling, but this release uses the smaller two-site exception patch already sufficient to enter LiteLLM's existing truncated-body fallbacks.

This keeps the fork delta minimal and makes removal straightforward after an upstream release contains an accepted equivalent fix.

## 6. Regression test

### 6.1 Test name and placement

Add the test near the existing routing-peek tests in `test_mcp_server.py`:

```python
test_mcp_routing_peek_survives_multibyte_char_split_at_cap
```

### 6.2 Payload construction

Build a complete, valid JSON-RPC `tools/call` request from ASCII prefix and suffix bytes, padding, and one literal multibyte character such as `é`.

The first byte of `é` must be at index `peek_cap - 1`, so the 4096-byte prefix contains only the first byte of the two-byte sequence:

```python
peek_cap = mcp_server._MCP_ROUTING_PEEK_MAX_BYTES
prefix = b'{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"progress_test","arguments":{"text":"'
suffix = b'"}}}'
padding = b"x" * (peek_cap - len(prefix) - 1)
body = prefix + padding + "é".encode("utf-8") + suffix
```

The test must assert its own preconditions:

- `len(body) > peek_cap`.
- `json.loads(body)` succeeds.
- `body[:peek_cap].decode("utf-8")` raises `UnicodeDecodeError`.

### 6.3 Request execution

Exercise `handle_streamable_http_mcp()` through the same ASGI seam used by the adjacent routing tests:

- Use a no-session `POST` request.
- Mock `extract_mcp_auth_context()` and session-manager initialization consistently with nearby tests.
- Provide the complete body through `receive`.
- Let the stateless handler drain the wrapped `receive` and capture every byte it receives.
- Make the stateful handler fail immediately if invoked.

### 6.4 Required assertions

The regression test passes only when all of the following are true:

1. The stateless manager is called exactly once.
2. The stateful manager is not called.
3. The bytes reconstructed by the stateless manager equal the original complete `body` exactly.
4. LiteLLM does not emit an HTTP 500 response start.
5. The request does not raise `UnicodeDecodeError` to the test caller.

One integration-style test covers both patched exception handlers sequentially: the initialize check must tolerate the truncated code point, and the later JSON-RPC response check must tolerate the same prefix before dispatch can complete.

## 7. Validation

Run the narrow checks first:

```bash
uv run pytest \
  tests/test_litellm/proxy/_experimental/mcp_server/test_mcp_server.py \
  -k multibyte_char_split_at_cap -q

uv run ruff check \
  litellm/proxy/_experimental/mcp_server/server.py \
  tests/test_litellm/proxy/_experimental/mcp_server/test_mcp_server.py

uv run ruff format --check \
  litellm/proxy/_experimental/mcp_server/server.py \
  tests/test_litellm/proxy/_experimental/mcp_server/test_mcp_server.py
```

Then run the full mapped MCP server test file:

```bash
uv run pytest \
  tests/test_litellm/proxy/_experimental/mcp_server/test_mcp_server.py -q
```

Finally, build the database image locally for the native platform:

```bash
docker build \
  -t litellm-database:mcp-utf8-ci \
  -f docker/Dockerfile.database .
```

The regression test must fail on the unpatched `v1.98.0` baseline with `UnicodeDecodeError` and pass after the two-site patch.

## 8. Fork-specific publication CI

### 8.1 Workflow ownership

Add one dedicated workflow:

```text
.github/workflows/publish-litellm-database.yml
```

Do not repurpose `.github/workflows/create-release.yml`; it creates GitHub releases and documents the upstream non-database image. Do not add publication behavior to `.circleci/config.yml`; its `build_docker_database_image` job currently builds a local CI artifact consumed by tests and does not own registry publication.

### 8.2 Trigger and repository guard

The publication path must:

- Trigger only when the fork-only source tag `v1.98.0-mcp-utf8.1` is pushed.
- Run only when `github.repository == 'conradsheeran/litellm'`.
- Use minimal permissions:

```yaml
permissions:
  contents: read
  packages: write
```

An optional `workflow_dispatch` path may perform a build-only dry run with `push: false`. It must not publish, and no manual input may select a different repository, Dockerfile, image name, or release tag.

### 8.3 Pre-publication gates

Before registry login or image push, the workflow must verify:

1. `v1.98.0` resolves to `d8f71d7bdbd7c9873d98293f83d64c6db72847e6`.
2. The current commit is tagged `v1.98.0-mcp-utf8.1` and descends from `v1.98.0`.
3. `server.py` contains zero old exception patterns and exactly two patched patterns.
4. The changed-file set from `v1.98.0` is limited to the MCP runtime file, its mapped test file, this publication workflow, and repository documentation for this hotfix.
5. The focused regression, full mapped MCP test file, Ruff check, and Ruff format check pass.

The workflow must stop before publication if any gate fails.

### 8.4 Build and push

Use `docker/Dockerfile.database` with repository root as build context.

Publish a multi-platform OCI image matching the upstream v1.98.0 platform set:

```text
linux/amd64
linux/arm64
```

Use Buildx/QEMU and pin all GitHub Actions dependencies to full commit SHAs, consistent with the repository's existing workflow security style.

The only allowed push target is:

```text
ghcr.io/conradsheeran/litellm-database:v1.98.0
```

Required build metadata:

```text
org.opencontainers.image.source=https://github.com/conradsheeran/litellm
org.opencontainers.image.revision=<patched commit SHA>
org.opencontainers.image.version=v1.98.0
org.opencontainers.image.title=litellm-database
org.opencontainers.image.description=LiteLLM v1.98.0 database image with MCP routing UTF-8 hotfix
```

Enable BuildKit provenance attestations. Preserve the Dockerfile's pinned base-image digests and do not inject alternate base images through CI build arguments.

### 8.5 Publication exclusions

The workflow must contain no push target matching any of these:

```text
ghcr.io/conradsheeran/litellm
ghcr.io/conradsheeran/litellm-non_root
ghcr.io/conradsheeran/litellm-slim
docker.io/*
```

It must not create a GitHub release, publish a Python package, update a Helm chart, or modify a package version.

### 8.6 Post-publication verification

After push, the workflow must inspect the published manifest and fail if the tagged OCI index does not contain both `linux/amd64` and `linux/arm64` image manifests.

The workflow must also:

- Report the top-level immutable digest in `$GITHUB_STEP_SUMMARY`.
- Pull or address the `linux/amd64` child manifest and verify that Python can import the installed LiteLLM MCP server module.
- Verify from the installed source that the old exception pattern count is zero and the patched pattern count is two.
- Confirm that no additional image tag was generated by metadata defaults.

The deployment reference should be recorded with the resulting digest after publication:

```text
ghcr.io/conradsheeran/litellm-database:v1.98.0@sha256:<published-index-digest>
```

## 9. Deployment acceptance

Change only the LiteLLM service image reference. Do not change routing, credentials, database configuration, Nginx, MCP Server configuration, or NPM clients.

Acceptance sequence:

1. Deploy the published digest to the LiteLLM service only.
2. Confirm the container starts and existing health checks pass.
3. Resubmit the same monthly Semantic Evidence Pack that previously failed at byte positions 4094-4095.
4. Confirm the LiteLLM 500 and UTF-8 decode error are absent.
5. Confirm the downstream MCP tool receives the complete payload and returns the normal tool result.

The production request chain must remain unchanged.

## 10. Rollback

Rollback requires only restoring the previous LiteLLM image reference and recreating that service. No database migration or data rollback is required.

Keep the pre-deployment image reference and digest in the deployment record. If acceptance fails:

```text
1. Restore the previous image reference.
2. Recreate only the LiteLLM service.
3. Verify health and one small MCP tool call.
```

Do not delete the fork image during rollback; retain its digest and CI logs for diagnosis.

## 11. Risks and controls

### Invalid UTF-8 requests

A genuinely invalid UTF-8 request may now proceed past routing classification and fail later in normal JSON parsing rather than escaping from the routing peek as an internal 500. This is acceptable: the routing peek must not decide validity of the full request, and downstream parsing remains authoritative.

### Fork drift

The hotfix is intentionally pinned to v1.98.0. Do not merge unrelated upstream changes into the hotfix release. The changed-file allowlist and exact baseline check prevent accidental drift.

### Tag ambiguity

The upstream Git tag remains untouched. The fork source tag identifies the patched commit, while the published image keeps the upstream-compatible `v1.98.0` tag. OCI revision labels and the recorded digest bind the image to the actual patch source.

### Upstream convergence

Issues #32226 and #34917 and PRs #32253 and #34919 track equivalent upstream work. Before a future LiteLLM upgrade, check whether the selected upstream release already handles `UnicodeDecodeError` at both routing parse sites. If it does, remove the fork patch rather than layering a duplicate fix.

## 12. Completion criteria

Implementation is complete only when:

- The fork patch is based on the exact `v1.98.0` commit.
- The two old exception handlers are replaced and no unrelated runtime behavior changes.
- The boundary-split regression is red on baseline and green after patch.
- The full mapped MCP server test file and formatting/lint checks pass.
- The dedicated workflow publishes only `ghcr.io/conradsheeran/litellm-database:v1.98.0`.
- The published OCI index contains both required Linux architectures and records the patched source revision.
- Production reproducer succeeds through the unchanged LiteLLM Gateway chain.
- Rollback information, image digest, source commit, and CI run URL are recorded.

## 13. References

- Upstream issue #34917: <https://github.com/BerriAI/litellm/issues/34917>
- Upstream issue #32226: <https://github.com/BerriAI/litellm/issues/32226>
- Upstream PR #34919: <https://github.com/BerriAI/litellm/pull/34919>
- Upstream PR #32253: <https://github.com/BerriAI/litellm/pull/32253>
- Upstream database image: <https://github.com/BerriAI/litellm/pkgs/container/litellm-database>
