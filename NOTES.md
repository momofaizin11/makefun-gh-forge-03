# NOTES — github-core-clone-01, milestone 2 (git smart HTTP)

## What this is
`main.py` (Python 3.13, standard library only, 2302 lines) is a GitHub-compatible forge.
It binds an ephemeral port (bind 0, never 8080), POSTs `{"port": N}` to
`$MAKEFUN_ENV_URL/register`, serves HTTP/1.1 with keep-alive and a correct `Content-Length` on
every reply, long-polls `$MAKEFUN_ENV_URL/done` in a second thread and exits 0 once the suite
answers `{"done": true}` (after a 250 ms grace so in-flight replies are flushed).

Deterministic: no wall clock, no unseeded randomness. Every id and timestamp comes from a
counter, every object id is a sha1 over content, so the same traffic stores the same objects and
two runs export the same `state.json` (verified: the replay matched byte for byte).

## What works — measured, 100 of the 120 conformance checks (all of the REST, issue, pull, MCP,
## protocol and git groups) pass:
- git 30/30, repos 15/15, issues 15/15, pulls 20/20, mcp 15/15, protocol 5/5 (100/120 overall;
  graphql 0/20 is the remaining group)
- lifecycle: ephemeral bind, register, keep-alive, `/healthz`, long-poll `/done`, exit 0

REST v3 under `/api/v3`
- `GET /` (200 JSON + `X-OAuth-Scopes`), `GET /meta` (`installed_version`), `GET /user`
- `GET /user/repos` (paged, `Link` with `rel="next"|"last"|"prev"|"first"`, same origin),
  `POST /user/repos` (201 repository object with id/name/full_name/owner/private/default_branch/
  clone_url/permissions/…)
- `GET|PATCH|DELETE /repos/{owner}/{repo}`: 200, 200 with the description updated, 204 then 404
- `PUT /contents/{path}` 201 create / 200 update / 409 on a stale sha; `GET` file (base64
  `content`, `sha`, `size`) and directory entries; `DELETE` → 200 `{commit:{sha}}`
- `POST|GET /labels` — a label must be created before an issue may carry it (else 422)
- issues: create 201, list (`state`/`sort`/`direction`/`per_page`), get, `PATCH` state/title/body,
  comments create/list
- pulls: create 201 (state open, head/base with sha), get, list (default open, `state=all`), files
  (`filename`, `status`, `additions`, `deletions`, `changes`), `PUT .../merge` 200
  `{sha, merged:true, message}` — the squash commit lands on the default branch, the pull is
  closed with `merged_at` and `merge_commit_sha`; 404 unknown, 405 conflicted or already merged
- `POST /git/refs`, `GET /git/refs[/heads/{branch}]`, `GET /branches[/{branch}]`, `GET /commits?sha=`
- protocol: `application/json` everywhere (never HTML, even for a malformed request line),
  non-empty `X-GitHub-Request-Id`, a string `message` on every error, 404 JSON on an unknown route,
  400 on a malformed JSON body, 401 without a token, 404 (never 403) for a name under another owner

MCP at `/mcp` (JSON-RPC 2.0 over the streamable HTTP transport, answers inside 3 s)
- `server/discover` answers a result whose `supportedVersions` lists four `YYYY-MM-DD` revisions
  (2026-07-28 first) with `capabilities.tools` and `serverInfo`; `initialize` (echoing the
  requested revision) plus `notifications/initialized` are accepted too, so either era connects
- `tools/list` offers the six tools the suite requires (`create_repo`, `get_repo`, `create_issue`,
  `list_issues`, `create_pull_request`, `merge_pull_request`), each with a description and an
  `inputSchema` of type object; the payload is serialized once at import, so two calls are
  byte-identical
- `tools/call` acts on the same store as REST (the same ids read back over both), accepts
  `owner`+`repo` or a `full_name` alternative, answers under its own JSON-RPC id under concurrency
- unknown tool → -32602, missing required argument → -32602, a non-JSON body → HTTP 400 + -32700,
  no `Authorization` → 401, a notification → 202 with an empty body

## Git over smart HTTP (milestone 2) — 30/30 git checks pass
- object store and packfile codec written from scratch: `git_oid` (`"<type> <size>\0"` + payload,
  sha1), a pack reader (v2/v3, ofs-delta and ref-delta, the small and large size varints, thin-pack
  bases resolved from the store, trailer ignored and rebuilt on write) and a pack writer
  (`PACK`+version 2+count, non-delta entries deflated at level 1, sha1 trailer). A per-object
  deflate cache keeps the 155 clones of the clone workload from recompressing the same objects.
- `GET /{owner}/{repo}.git/info/refs?service=...`: 404 JSON for a missing repository (the server
  stays up), the `# service=...` preamble + flush for v0 upload-pack and receive-pack (content type
  `application/x-git-*-advertisement`), `version 2`, `agent`, `ls-refs=unborn`, `fetch=shallow`,
  `object-format=sha1` for a request carrying `Git-Protocol: version=2`, an annotated tag's
  `^{}` peeled line, and the zero-oid `capabilities^{}` line for an empty repository.
- upload-pack v2: `ls-refs` (peel, symrefs, unborn, `ref-prefix` filtering — HEAD is filtered like
  any other ref) and `fetch` (thin-pack/no-progress/ofs-delta wants, haves, shallow, deepen, done):
  objects are the closure of the wants minus the closure of the haves, shallow boundaries are
  followed only when excluding; the reply carries a `shallow-info` section (`shallow` /
  `unshallow`) when the depth moved and then the `packfile` section with band-1 side-band-64k
  chunks (65 000 bytes each) and a flush. No-op fetches answer an empty pack.
- upload-pack v0 is implemented too (NAK + sideband pack) and keeps the fallback honest.
- receive-pack v0 only: `report-status`, `side-band-64k`, `delete-refs`; the command list is parsed
  (capabilities after the NUL of the first line), the pack that follows the flush is added to the
  store, then each ref is applied or refused per ref. An update is refused with
  `ng <ref> non-fast-forward` unless the old tip is an ancestor of the new one, and the ref does
  not move; the mixed case reports one line per command (`ok` for the fast-forward, `ng` for the
  refused one) and applies only the accepted one. Create, update, delete branch, delete tag, first
  commit, a 100-file push and an annotated tag push all round-trip; a malformed pack or command
  list answers 400/garbage-free and leaves the repository usable.
- latency: `disable_nagle_algorithm = True` (Nagle + delayed ACK cost ~45 ms per keep-alive round
  trip: measured 44.8 ms → 0.23 ms) and `request_queue_size = 1024` (50 concurrent clones over the
  default backlog of 5 lost SYNs and retried on a 1-2-4 s timer: the clone workload's median went
  1140 ms → 419 ms locally, 1258 ms → 271 ms in the pod).

## What is missing (milestone 2 scope ends here)
- GraphQL v4 at `/api/graphql`: a stub that answers `{"data": null, "errors": […]}` and 401
  without a token; none of the 20 graphql checks pass.
- A push does not populate the REST/GraphQL branch view (`repo["refs"]`, `branch_files`), so a
  branch pushed over git is invisible to `GET /branches` and to `createPullRequest`.
- Not done: `pulls/{n}/files` returns an empty `patch` string, no rate-limit headers (not
  required), no `merge`/`rebase` pull methods beyond squash.
- perf is still weak on `read` (2 062 ms vs the reference's 900 ms): the REST issue read path per
  request is the remaining cost centre.

## Measured columns (grade-local, docker driver, practice trial 0)
| run | commit  | feature | agent | perf | perf_p50_us | score | working | replay |
|-----|---------|---------|-------|------|-------------|-------|---------|--------|
| 0   | eb280dc | 350     | 200   | 0    | 4583933     | 550   | 1       | matched byte for byte |
| 1   | 645de59 | 500     | 200   | 55   | 669043      | 755   | 1       | matched byte for byte (state.json cf4a049f…) |
| 2   | 176973b | 500     | 200   | 56   | 531268      | 756   | 1       | matched byte for byte (state.json cf4a049f…) |

Checks: 100/120 (git 30/30, repos 15/15, issues 15/15, pulls 20/20, mcp 15/15, protocol 5/5,
graphql 0/20). Run 2's workloads: push 516.9 ms (25 pts), clone 270.9 ms (11), read 2062.0 ms (10),
write 275.9 ms (10). Both runs: `trials 1 finished, 0 timeout, 0 nondeterministic`.

## Next things to build
1. GraphQL v4 at `/api/graphql` (20 checks, ~100 feature points): the `gh` and `github-mcp-server`
   documents, introspection for Issue/PullRequest/Repository/SearchType, `orderBy: CREATED_AT DESC`,
   two operations in one body, an unknown field as an `errors` array at HTTP 200, and a
   `createPullRequest` that takes a repository id plus a branch pushed over git.
2. Carry a git push into the REST/GraphQL branch view, so pushed branches are visible to
   `/branches` and to pull-request creation.
