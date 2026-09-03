# GitHub API response-shape contract

Derived from what the Mintlify server (`~/Developer/server`) actually reads. Every field listed
below is dereferenced somewhere in the codebase; fields *not* listed can be omitted or stubbed.
File references are `path:line` in that repo.

## Summary

| Endpoint | 7d prod | Primary callers | Tier |
|---|---:|---|---|
| `PATCH /repos/{o}/{r}/check-runs/{id}` | 1.7M | `GithubCheck.update/skip/close`, `GithubSourceCheck.update/close` | 1 |
| `GET /repos/{o}/{r}/contents/{path}` | 1.58M | `GitService.getFileBufferByPath`, `getContentDirectorySha`, `fetchConfig`/`fetchMintIgnore`/`fetchAssistantMd`, `$ref` resolution | 1 |
| `GET /repos/{o}/{r}/git/trees/{sha}` | 591k | `fetchFullGitTree` via `getContentTree`; onboarding docs.json scan; template seeding | 1 |
| `GET /repos/{o}/{r}/compare/{basehead}` | 457k | `compareRef`, `compareCommits`, `getFilteredContentTreeDiff`, `getBranchInfo` | 1 |
| `GET /repos/{o}/{r}/commits[/{ref}]` | 398k | `getLatestCommit`, ref-existence probes | 1 |
| `GET /repos/{o}/{r}/git/blobs/{sha}` | 312k | `getBlob`, `getFileBufferBySha`, template blob download | 1 |
| `POST /app/installations/{id}/access_tokens` | 148k | implicit — `App#getInstallationOctokit`, `AppAuthProvider` | 1 |
| `POST /repos/{o}/{r}/check-runs` | 144k | `GithubCheck.create/createSkipped`, `GithubSourceCheck.create` | 1 |
| `GET /repos/{o}/{r}/pulls/{n}` | 89k | PR summary/status, merge follow-up, `read_prs`, writer tools | 3 |
| `POST /graphql` | 83k | commit push, latest-oid, branches, repos, blame, search, PR mutations | 2 |
| `GET /repos/{o}/{r}` | 72k | merge method, default branch, permissions, privacy, access probe | 1 |
| `GET /repos/{o}/{r}/commits` (list) | 48k | `getLatestCommit`, writer `list_commits` | 1 |
| `GET /repos/{o}/{r}/branches/{branch}` | 14k | `getBranchProtections`, `getBranchHeadSha` | 3 |
| `PUT /repos/{o}/{r}/pulls/{n}/merge` | 12k | `mergePr` | 3 |
| `POST /repos/{o}/{r}/deployments` + `/statuses` | 8k | preview deployments | 4 |
| `GET /repos/{o}/{r}/rules/branches/{branch}` | 5.7k | `getBranchProtections`, commit-message regex | 3 |
| `GET /repos/{o}/{r}/pulls` | 5k | `listOpenPullRequests`, `getPullRequestStatus`, `findOpenPrForRef` | 3 |
| `GET /app/installations/{id}` | — | `getPermissionsByInstallationIdV2`, ownership verify | 4 |
| `GET /installation/repositories` | — | installation repo list (paginated) | 4 |
| `POST` / `DELETE /repos/{o}/{r}/git/refs[/{ref}]` | tail | create/delete branch | 2 |
| `POST /repos/{o}/{r}/pulls` | tail | `createPr` | 3 |
| `POST /repos/{t}/{t}/generate` | tail | onboarding template seeding | 4 |
| `GET /repos/{o}/{r}/commits/{sha}/pulls` | tail | `getBranchInfo` | 3 |
| `GET`/`POST /repos/{o}/{r}/pulls/{n}/reviews` | tail | review status, approve/request-changes | 3 |
| `GET`/`POST /repos/{o}/{r}/issues/{n}/comments` | tail | PR comment upsert, comment bot | 3 |
| `GET /user`, `GET /users/{u}` | tail | OAuth identity, scopes header, Slack mapping | 4 |
| `GET /repos/{o}/{r}/collaborators/{u}/permission` | tail | write-access check | 3 |
| `GET /repos/{o}/{r}/git/ref/{ref}` | tail | branch-exists probe, head sha wait | 2 |
| `POST git/blobs`, `POST git/trees`, `POST git/commits`, `PATCH git/refs/{ref}` | tail | template seeding, repo copy | 4 |
| `POST /repos/{o}/{r}/merges` | tail | `mergeBranch` (bypass only) | 3 |
| `GET /repos/{o}/{r}/readme` | tail | repo description, docs generation | 4 |
| `GET .../zipball/{ref}`, `.../tarball/{ref}` | tail | hosted repo export, template seeding | 4 |
| `GET /search/issues` | tail | workflow PR search | 4 |
| `GET /repos/{o}/{r}/pulls/{n}/files` | tail | changed-file listing, comment sync | 3 |
| reactions create/delete | tail | comment bot 👀 / 👍 | 4 |
| Webhooks (`/github-webhook`) | — | see [Webhooks](#webhooks) | 5 |

Tiers: 1 = deploy reads, 2 = editor writes, 3 = PR flow, 4 = accept-and-forget stubs, 5 = webhooks.

## Cross-cutting

**Two clients.** `octokit` (`api/clients/octokit.ts`) with `@octokit/plugin-throttling` (one retry
on primary *and* secondary rate limits) plus a metrics plugin, and a hand-rolled fetch client
`GitHubApiClient` (`api/clients/githubApiClient.ts`) used by `GithubBypassService`. GHES instances
get `OctokitWithThrottling.defaults({ baseUrl })` (`api/utils/github/enterpriseRouters.ts:45`).

**Request headers sent by the bypass client** (`githubApiClient.ts:441-451`):
`Authorization: bearer <token>`, `Content-Type: application/json`,
`User-Agent: mintlify-server`, `Accept: application/vnd.github.v3+json`.
Path params are `encodeURIComponent`'d; for `GET`, every remaining option becomes a query param
(`githubApiClient.ts:353-380`).

**Response headers actually read:**

| Header | Where | Notes |
|---|---|---|
| `x-ratelimit-limit`, `x-ratelimit-remaining`, `x-ratelimit-reset` | `utils/github/rateLimitMetrics.ts:58-62`; `githubApiClient.ts:281-297` | **all three or nothing** — metrics silently skip otherwise. `reset` is epoch **seconds**. `remaining === 0` makes the bypass client sleep `retry-after + 1` (default 61s) and retry once. |
| `x-ratelimit-used`, `x-ratelimit-resource` | `rateLimitMetrics.ts:63-65` | optional; default `limit - remaining` and `"unknown"`. |
| `retry-after` | `githubApiClient.ts:285`; `rateLimitMetrics.ts` | seconds. |
| `x-github-request-id` | `githubApiClient.ts:152,164` | `probeRepoAccess` only. |
| `x-github-sso` | `utils/github/errors.ts:34-40` | on 403: value starting with `required` → SAML; `url=(\S+)` extracted as `authorizationUrl`. |
| `x-oauth-scopes` | `onboarding-github-repo.service.ts:158` | on `GET /user`; comma-split; `repo` ⇒ can access private. **Load-bearing.** |
| `content-type` | `githubApiClient.ts:400`, `githubBypass.service.ts:1130` | error bodies must be `application/json` or the body is dropped. Raw media must respond `application/vnd.github.raw...`. |
| `content-length` | `githubApiClient.ts:493`; `githubBypass.service.ts:1139` | `0` ⇒ `{data: null}`; used for the media size guard. |
| `link` | `hostedRepo.service.ts:447`; every `octokit.paginate` call | `rel="last"` + its `page=` query param drives the org repo count. |

`ETag` / `If-None-Match` are **never** used. No conditional requests, no 304 handling.

**Error envelope.** Non-2xx must be JSON with a top-level `message` string. The bypass client
reconstructs an octokit `RequestError` from `statusText` + `status` + parsed body
(`githubApiClient.ts:388-427`). A `401` with an auth fallback configured transparently retries the
whole request through installation auth (`githubApiClient.ts:483-489`).

String-matched messages (exact substrings the server branches on):

| String | Status | Where |
|---|---|---|
| `No commit found for the ref` | 404 | `utils/github/errors.ts:74` → `ref_not_found` |
| `Reference already exists` (**exact equality**) | 422 | `githubBypass.service.ts:2314` → duplicate-branch 409 |
| `A pull request already exists` (inside `errors[].message`) | 422 | `validators/git.validators.ts:69` |
| `No commits between ` (inside `errors[].message`) | 422 | `validators/git.validators.ts:97` |
| `your own pull request` (case-insensitive, inside `errors[]`) | 422 | `validators/git.validators.ts:83` |
| `Update is not a fast forward` | 422 | `github.service.ts:1592` |
| `Resource not accessible by integration` | 403 | `github.service.ts:1601` |
| `IP allow list` | 403 | `utils/github/errors.ts:42` |
| `/template repository/i`, `/is not a template/i` | 422 | `onboardingTemplateSeed.service.ts:741` |
| `/repository limit\|repository_limit\|maximum number of repositories/i` | 403/422 | `utils/hostedRepoOrg.ts:14` |
| `/secondary rate limit\|abuse detection/i` | 403/429 | `rateLimitMetrics.ts` classification |

The `422` validator envelope must be `{"errors": [{"resource": "...", "code": "...", "message": "..."}]}`
for the PR cases (`selfReviewError` also accepts bare strings in `errors[]`).

**Bot identities** the server matches on (`utils/github/githubBotName.ts`):
`mintlify[bot]` / `109931778+mintlify[bot]@users.noreply.github.com`, and
`mintlify-development[bot]` / `109878554+mintlify-development[bot]@users.noreply.github.com`.

---

# Tier 1 — deploy reads

## PATCH /repos/{owner}/{repo}/check-runs/{check_run_id}

**1.7M/7d — the single hottest call.**

Call sites: `api/workers/workflows/updateWorkflow/statusReporters/GithubCheck.ts:165` (`update`),
`:180` (`skip`), `:198` (`close`); `api/workers/workflows/sourceChecks/statusReporters/GithubSourceCheck.ts:71`
(`update`), `:114` (`close`).

**Request** — `owner`, `repo`, `check_run_id` in path; body is one of:

- `update`: `{status: "in_progress", output: {title, summary, text}}` — title `"Deploying your docs..."`
  (`"Validating your docs..."` for source checks, plus `output.annotations`).
- `skip`: `{status: "completed", conclusion: "skipped", output: {title: "Deployment Skipped", summary, text: ""}}`.
- `close`: `{status: "completed", details_url, conclusion, output: {title, summary, text}}`.
  `conclusion` is `"success" | "failure"`; `GithubSourceCheck` remaps a `failure` by notification
  level — `disabled → "skipped"`, `warning → "neutral"`, `blocking → "failure"` (`GithubSourceCheck.ts:10-15`).

No `Accept` or `X-GitHub-Api-Version` header is set on check-run calls.

**Response fields read: none.** The body is discarded entirely; only the HTTP status matters.
Return `200` with any JSON object.

**Error behavior.** On GHES only (`octokit.request.endpoint.DEFAULTS.baseUrl !== "https://api.github.com"`),
`404` and `401` are retried up to 3 attempts with a fixed 750 ms delay (`GithubCheck.ts:16-39`).
Every other status rethrows immediately. `GithubSourceCheck` has no retry.

**Fidelity notes.**
- One check run is created per deployment then patched on *every* workflow step boundary
  (`Workflow.ts:45,68,132,359`) with exactly one terminal `close` (`:432`/`:449`). Between PATCHes
  only `output.summary` and `output.text` change — `text` is the *cumulative* log array re-serialized
  in full each time, so payloads grow monotonically toward the cap.
- `output.summary` and `output.text` are both truncated client-side at
  `GITHUB_CHECK_MAX_TEXT_LENGTH = 65535` with a `"\n\n…output truncated"` marker; `output.title` is
  sliced to `GITHUB_CHECK_MAX_TITLE_LENGTH = 1024` (`GithubCheck.ts:6,43,47-67`).
- Annotations are chunked at **50 per request** and the chunks are sent as **parallel PATCHes to the
  same `check_run_id`** via `Promise.all` (`GithubSourceCheck.ts:69,110-121`). A fake server must
  accept concurrent PATCHes and accumulate annotations rather than replace them. With no
  annotations, a single PATCH carries `annotations: undefined`.
- Annotation shape sent (`api/types/workflows/annotation.ts`):
  `{path, message, start_line, end_line, annotation_level: "notice"|"warning"|"failure", title?}`.
  No `start_column`/`end_column`/`raw_details`.

## GET /repos/{owner}/{repo}/contents/{path}

**1.58M/7d.** Two distinct call shapes — the endpoint returns an **object for files and an array for
directories**, and both branches are exercised.

Call sites: `api/services/github.service.ts:915` and `api/services/githubBypass.service.ts:1211`
(`getFileBufferByPath`); `github.service.ts:1221` / `githubBypass.service.ts:1545`
(`getContentDirectorySha`); `githubBypass.service.ts:1107` (`getMediaStreamByPath`, raw);
`api/services/hostedRepo.service.ts:193` (`waitForRepoFileReady`).

**Request params:** `owner`, `repo`, `path`, `ref` (= `this.sha ?? this.ref` for file reads,
`this.ref` for the directory read). No `mediaType` on the octokit path. The bypass client's
`getMediaStreamByPath` and `getFileBufferBySha` send `Accept: application/vnd.github.raw+json`.

**File response — minimal object with exactly the read fields:**

```json
{
  "type": "file",
  "sha": "a94a8fe5ccb19ba61c4c0873d391e987982fbbd3",
  "size": 1342,
  "content": "IyBIZWxsbwo=\n",
  "encoding": "base64"
}
```

Consumption order (`github.service.ts:984-1018`):
1. `Array.isArray(data)` ⇒ `"expected path to point to file; got directory"`.
2. `data.type !== "file"` ⇒ `expected path to point to file; got "<type>"`. `type` must be
   exactly `"file"`.
3. `!data.content && data.size > 0` ⇒ falls back to `GET /git/blobs/{data.sha}`. This is the >1 MB
   path — `size` and `sha` are both load-bearing.
4. Otherwise `Buffer.from(data.content || "", "base64")`. **`encoding` is never inspected** — base64
   is assumed unconditionally. `data.sha` becomes `uniqueId` and is surfaced as the `X-Unique-Id`
   response header (`git.controller.ts:304`).

`download_url`, `name`, `path`, `html_url`, `git_url`, `url`, `_links` are unread on the file path.

**Directory response — array; only `path` and `sha` are read:**

```json
[
  { "path": "docs", "sha": "8e2f3d1a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e" },
  { "path": "README.md", "sha": "1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b" }
]
```

`getContentDirectorySha` requires an array (a non-array throws using `.type` in the message), finds
the entry whose `path` exactly equals `gitSource.contentDirectory`, and returns its `sha`. A miss
throws `Could not find sha for contentDirectory ...` — a string the controller pattern-matches to a
404 (`git.controller.ts:424,483`).

**Error behavior.**
- `404` + message containing `No commit found for the ref` ⇒ `reason: "ref_not_found"`.
- Plain `404` ⇒ the server re-probes `GET /repos/{owner}/{repo}`. If *that* also 404s the summary
  becomes "Application does not have access to the repository"; otherwise `reason: "not_found"`.
- `403` ⇒ "The Mintlify installation was suspended…".
- `waitForRepoFileReady` (`hostedRepo.service.ts:193`) retries only on `404` — 5 attempts, 3 s
  apart; **any non-404 is treated as ready**.

**Fidelity notes.** The raw variant must respond with a `content-type` starting with
`application/vnd.github.raw` — `getMediaStreamByPath` destroys the stream and errors otherwise
(`githubBypass.service.ts:1130`) — and its `content-length` drives a `maxBytes` guard that tolerates
an absent or non-numeric value. The high call volume comes from config resolution: `docs.json`,
`mint.json`, `.mintignore` (twice, with and without content dir), `.mintlify/Assistant.md` /
`ASSISTANT.md`, and one request per `$ref` in the config (`api/interfaces/GitService.ts:162,202,305,333`).

## GET /repos/{owner}/{repo}/git/trees/{tree_sha}

**591k/7d.**

Call sites: `github.service.ts:734` / `githubBypass.service.ts:1024` (via `fetchFullGitTree`);
`api/services/onboarding-github-repo.service.ts:143`; `api/services/onboardingTemplateSeed.service.ts:367`;
`api/services/hostedRepo.service.ts:614`.

**Request params:** `tree_sha` (**may be a branch name, not only a SHA** —
`onboardingTemplateSeed.service.ts:372` passes `repo.default_branch`), and `recursive: "true"`
(string, not boolean) when recursion is wanted.

**Response — exactly the read fields:**

```json
{
  "sha": "4b825dc642cb6eb9a060e54bf8d69288fbee4904",
  "truncated": false,
  "tree": [
    { "path": "docs/index.mdx", "mode": "100644", "type": "blob", "sha": "0f1e2d3c4b5a69788796a5b4c3d2e1f0a9b8c7d6", "size": 2048 },
    { "path": "docs/images",    "mode": "040000", "type": "tree", "sha": "aa11bb22cc33dd44ee55ff6677889900aabbccdd" }
  ]
}
```

`toTreeNode` (`api/utils/git.ts:294-302`) requires `sha`, `path`, `mode`, `type` on **every** entry —
any one `undefined` throws `"Git tree entry is missing required fields; refusing to operate on a
partial file list"`. `size` is optional. `url` is never read.

**`truncated` handling — `fetchFullGitTree` (`api/utils/git.ts:328-368`):**
1. Fetch root with `recursive=true`. `truncated: false` ⇒ done in one request, `descended: false`.
2. Otherwise re-fetch the same sha with `recursive=false`. If a **non-recursive** listing still
   reports `truncated: true`, it **throws** `"Git tree listing for '<prefix|/>' was truncated even
   without recursion..."`. Every entry is recorded; each `type === "tree"` entry becomes a frontier
   item `{sha, prefix: "<path>/"}`.
3. BFS over the frontier, each subtree fetched with `recursive=true`, concurrency 8
   (`TREE_FETCH_CONCURRENCY`), depth ≤ 12 (`MAX_TREE_DESCENT_DEPTH`). Truncated subtrees are
   re-expanded shallowly.
4. Non-empty frontier after 12 levels ⇒ throws `"Git tree descent exceeded 12 levels..."`.

**Fidelity notes.**
- Recursive responses must return **repo-relative** paths; non-recursive responses must return
  **bare names** — `fetchFullGitTree` prepends the accumulated prefix itself.
- `filterContentTree` (`git.ts:44-81`) keeps only `mode` `100644`/`100755`; `120000` (symlink),
  `160000` (submodule) and `040000` are dropped from the file set.
- Other callers treat `truncated: true` as fatal rather than descending: onboarding throws
  `OnboardingGithubRepositoryTreeTooLargeError` → HTTP 422 "Repository is too large to inspect for
  docs.json"; template seeding falls back to tarball download.

## GET /repos/{owner}/{repo}/compare/{basehead}

**457k/7d.**

Call sites: `github.service.ts:136` (`compareCommits`, paged), `:1864` (`compareRef`, unpaged);
same in `githubBypass.service.ts:142` / `:2192`.

**Request params:** `basehead` as `"<base>...<head>"`; `compareCommits` also sends `per_page: 250`
and `page`. `compareRef` sends neither.

**Response — exactly the read fields:**

```json
{
  "status": "ahead",
  "ahead_by": 3,
  "merge_base_commit": { "sha": "c0ffee1234567890abcdef1234567890abcdef12" },
  "commits": [
    {
      "sha": "1111111111111111111111111111111111111111",
      "commit": {
        "message": "docs: fix typo",
        "author": { "name": "Ada Lovelace" },
        "committer": { "date": "2026-08-30T10:15:00Z" }
      },
      "author": { "login": "ada" }
    }
  ],
  "files": [
    { "filename": "docs/index.mdx", "status": "modified", "sha": "0f1e2d3c4b5a69788796a5b4c3d2e1f0a9b8c7d6" },
    { "filename": "docs/new.mdx",   "status": "renamed",  "sha": "aabbccddeeff00112233445566778899aabbccdd", "previous_filename": "docs/old.mdx" }
  ]
}
```

Field-by-field:
- `status` — narrowed to `ahead`/`behind`/`identical`/`diverged`; anything else ⇒ `"unknown"`
  (`github.service.ts:154-162`).
- `ahead_by` — `> 0` ⇒ `hasNewCommits` (`:1918`).
- `merge_base_commit.sha` — **non-optional**; dereferenced with no null guard (`:810`).
- `commits[]` — empty array terminates pagination (`:220`); `commits.at(-1)` is treated as the head
  commit, reading `.sha` and `.commit.committer?.date` (nullable). `commits[last].sha === head.sha`
  also terminates pagination (`:227`). `editorBranch.service.ts:517-527` additionally reads
  `commits[].author?.login` and `commits[].commit.author?.name` (both nullable; author-less commits
  are dropped).
- `files[]` — `.filename` (map key for pagination dedupe *and* `GitTreeDiff.path`), `.status`
  (`copied → added`, `changed → modified`, else passed through as `added`/`modified`/`removed`/`renamed`),
  `.sha` (→ `uniqueId`), `.previous_filename` (→ `previousPath`). `files` may be absent; a missing
  `files` on the diff path is a hard error that then probes whether `base` is a missing ref (`:787`).

**Not read anywhere:** `behind_by`, `total_commits`, `base_commit.*`, `url`/`html_url`/`permalink_url`/
`diff_url`/`patch_url`, and per-file `patch`, `additions`, `deletions`, `changes`, `raw_url`,
`contents_url`, `blob_url`. Per-commit `html_url`, `commit.author.email/date`, `author.avatar_url`
are also unread.

**Error behavior.** `compareRef` swallows everything and returns `null`. `compareRefs` maps a `404`
into two follow-up `GET /commits/{ref}` probes to decide `base_missing` vs `head_missing` vs
`unknown` (`github.service.ts:165-190`).

**Fidelity notes.** GitHub caps `files[]` at 300 per page; the server pages with `per_page: 250` up
to `MAX_PAGINATION_PAGES = 25` and dedupes by filename (`github.service.ts:206-235`) — note the
bypass clone dedupes by `file.sha` instead (`githubBypass.service.ts:228`), a latent divergence.

## GET /repos/{owner}/{repo}/commits and /commits/{ref}

**398k + 48k/7d.**

Call sites: `github.service.ts:658` and `:704` (list, `sha: ref`), `:684`
(`/commits/{commit_sha}`), `:168` and `:1882` (`/commits/{ref}` existence probes);
`api/services/writerService/tools/listCommits.ts:90` (paged list).

**Request params:** list — `sha` (branch or commit ref), plus `since`, `until`, `per_page`, `page`
for the writer tool. Single — `commit_sha` or `ref` in the path.

**Response — list is an array; only `[0]` is read by the deploy path:**

```json
[
  {
    "sha": "1111111111111111111111111111111111111111",
    "url": "https://api.github.com/repos/acme/docs/commits/1111111111111111111111111111111111111111",
    "html_url": "https://github.com/acme/docs/commit/1111111111111111111111111111111111111111",
    "commit": {
      "message": "docs: add quickstart",
      "author":    { "name": "Ada Lovelace", "date": "2026-08-30T10:14:00Z" },
      "committer": { "date": "2026-08-30T10:15:00Z" }
    },
    "author": { "login": "ada", "name": "Ada Lovelace" }
  }
]
```

Read: `.sha`, `.commit.committer?.date` (nullable → `null`), `.url`, `.commit.message`
(`github.service.ts:672-677`). The writer tool additionally reads `.commit.author?.name`,
`.author?.name`, `.author?.login`, `.commit.author?.date`, `.html_url`
(`listCommits.ts:104-111`). An empty array throws `"No commits found"`.

The single-commit form returns the same object (not wrapped in an array) and the same four fields
are read.

**Error behavior.** `/commits/{ref}` is used purely as an existence probe — `404` ⇒ ref missing
(`github.service.ts:1889`); the body is never read.

**Fidelity note.** The writer tool pages with `per_page: 100` and stops when a page returns fewer
than `per_page` items — no Link header needed.

## GET /repos/{owner}/{repo}/git/blobs/{file_sha}

**312k/7d.**

Call sites: `github.service.ts:845` (`getBlob`), `:860` (`getFileBufferBySha`);
`githubBypass.service.ts:1063` (`getBlob`), `:1079` (`getFileBufferBySha`, **streamed**);
`onboardingTemplateSeed.service.ts:435`; `hostedRepo.service.ts:693`.

**Response:**

```json
{
  "sha": "0f1e2d3c4b5a69788796a5b4c3d2e1f0a9b8c7d6",
  "content": "IyBRdWlja3N0YXJ0Cg==\n",
  "encoding": "base64"
}
```

- `getBlob` returns the whole object; `githubUpdate.service.ts:105` reads `.content` and assumes
  base64 without checking `encoding`.
- `getFileBufferBySha` (octokit) reads `result.data.content || ""` and base64-decodes.
- `onboardingTemplateSeed.service.ts:441` **does** check `blob.encoding === "base64"` and throws
  `Unexpected Git blob encoding for <path>` otherwise.
- `hostedRepo.service.ts` passes `.content` straight through to `POST git/blobs`.

**Fidelity note.** The bypass client's `getFileBufferBySha` sends
`Accept: application/vnd.github.raw+json` and consumes the response as a **raw byte stream** with no
JSON parsing (`githubBypass.service.ts:1079-1091`) — the fake must honour that Accept and return
raw bytes there. The same route on the octokit client expects JSON+base64. Blob download during
template seeding runs 16 workers concurrently.

## POST /app/installations/{installation_id}/access_tokens

**148k/7d.** No direct call site — issued implicitly by `App#getInstallationOctokit(id)` and by
`AppAuthProvider.getTokenWithExpiry` (`api/clients/githubApiClient.ts:80-118`, `@octokit/auth-app`).

**Request body** (only when scoping is requested): `{repositories: [...], permissions: {...}}`.
`AppAuthProvider.getToken` skips its in-process cache whenever either is supplied
(`githubApiClient.ts:54-57`).

**Response — the standard envelope:**

```json
{
  "token": "ghs_16C7e42F292c6912E7710c838347Ae178B4a",
  "expires_at": "2026-09-02T18:00:00Z",
  "permissions": { "contents": "write", "pull_requests": "write", "checks": "write" },
  "repository_selection": "all"
}
```

`token` and `expires_at` are read (`githubApiClient.ts:101-104`); `expires_at` must parse via
`new Date(...)`. Tokens are cached per `installationId:baseUrl` and reused until within 5 minutes of
expiry (`EXPIRY_BUFFER_MS`), with in-flight de-duplication.

**Fidelity notes.** Token prefixes are classified for metrics only
(`utils/github/rateLimitMetrics.ts:43-51`): `ghs_` installation, `ghu_` user-to-server, `gho_` oauth,
`ghp_`/`github_pat_` PAT, `eyJ` app JWT. Mint failures are logged as
`github_app_installation_token_mint_failed` with the status. A `401` on any subsequent request with
an auth fallback configured causes a transparent re-auth through installation credentials.

## POST /repos/{owner}/{repo}/check-runs

**144k/7d.**

Call sites: `GithubCheck.ts:107` (`create`), `:144` (`createSkipped`); `GithubSourceCheck.ts:51`
(`create`).

**Request bodies:**
- `create`: `{name, status: "queued", head_sha, external_id}`. `name` defaults to
  `"Mintlify Deployment"`. `external_id` is the deployment-history id.
- `createSkipped`: `{name, status: "completed", head_sha, conclusion: "skipped", output: {title, summary, text}}`.
  Deliberately **no `external_id`**.
- `GithubSourceCheck.create`: `{name: "Mintlify Validation (<subdomain>) - <checkKey>", status: "queued", head_sha}`.
  No `external_id`, no `output`.

**Response — only `data.id` is read:**

```json
{ "id": 40217386 }
```

Destructured as `const { data: { id } } = ...` at all three sites and persisted as
`deploymentHistory.githubCheckId`. `html_url`, `node_id`, `status`, `conclusion`, `output`,
`check_suite` are never read.

**Error behavior.** Same GHES 404/401 3-attempt retry as the PATCH. Create failure is swallowed by
`createGithubCheckReporter.ts:34-45` into
`"Unable to create GitHub check. Is the GitHub app installed?"`. The push webhook creates skipped
checks under `Promise.allSettled` and drops rejections (`githubWebhooks.controller.ts:1164-1198`).

**Fidelity note.** The push handler fans out **one POST per commit** in a push (or per non-tracking
sha), so a fake must tolerate bursts of creates against distinct `head_sha` values.

## GET /repos/{owner}/{repo}

**72k/7d.** Called for many different single fields, so the response must be reasonably complete.

Call sites (26 total): `github.service.ts:386` (merge method), `:952` (access probe), `:1939`
(name), `:1953` (privacy), `:2003` (default branch), `:2094` (permissions);
`onboarding-github-repo.service.ts:123`; `onboardingTemplateSeed.service.ts:363`;
`hostedRepo.service.ts:156,531,577`; `repoDescription.service.ts:39`;
`api/workers/workflows/generateDocs/steps/discoverProjectContext.ts:17`; `githubApiClient.ts:142`
(`probeRepoAccess`).

**Response — union of every read field:**

```json
{
  "name": "docs",
  "full_name": "acme/docs",
  "private": false,
  "default_branch": "main",
  "description": "Acme product documentation",
  "homepage": "https://docs.acme.com",
  "topics": ["docs", "mintlify"],
  "stargazers_count": 42,
  "license": { "spdx_id": "MIT" },
  "owner": { "login": "acme" },
  "permissions": { "admin": false, "maintain": false, "push": true, "pull": true },
  "allow_squash_merge": true,
  "allow_merge_commit": true,
  "allow_rebase_merge": false
}
```

- Merge-method selection precedence: `allow_squash_merge → "squash"`, else `allow_merge_commit →
  "merge"`, else `allow_rebase_merge → "rebase"`, else `"merge"`; any throw also yields `"merge"`
  (`github.service.ts:394-400`).
- `permissions` absent or `permissions.push` falsy ⇒ `hasWriteAccess: false` (`:2102`). Onboarding
  additionally accepts `permissions.admin` or `permissions.maintain` (`onboarding-github-repo.service.ts:125-128`).

**Error behavior.** `404`/`403`/`401` ⇒ `{hasAccess: false, hasWriteAccess: false}`; any other status
rethrows (`github.service.ts:2109-2119`). `hostedRepo.service.ts:156` treats `404` as
"doesn't exist" and rethrows everything else. `probeRepoAccess` (`githubApiClient.ts:135-173`) never
throws — it returns `{status, message, requestId}` and reads `x-github-request-id` and
`response.data.message`, with a 5 s `AbortSignal.timeout`.

---

# Tier 2 — editor writes

## POST /graphql

**83k/7d.** All documents are string literals in `api/services/github.service.ts`,
`api/services/githubBypass.service.ts`, and
`api/services/deploymentService/gitUpdateService/githubUpdate.service.ts`. Both clients POST
`{query, variables}`; the bypass client sends `Accept: application/vnd.github.v3+json` and
`Content-Type: application/json` (`githubApiClient.ts:598-610`).

**Errors.** A `200` body containing a non-empty `errors[]` is turned into a `GraphqlResponseError`
(`githubApiClient.ts:676-690`), so the response must be
`{"data": null, "errors": [{"type": "...", "message": "...", "path": [...], "locations": [...]}]}`.
Only `errors[0].type` and `errors[0].message` are read.

**There is no retry keyed on GraphQL error messages.** The only GraphQL retry is the shared
rate-limit retry: `x-ratelimit-remaining: 0` on the response causes one sleep-and-retry of
`retry-after + 1` seconds, default 61 (`githubApiClient.ts:615-631`). Octokit's throttling plugin
adds one retry on primary and one on secondary rate limits.

### 1. `getLatestCommit` — `github.service.ts:608`, `githubBypass.service.ts:831`

```graphql
query getLatestCommit($owner: String!, $name: String!, $branch: String!) {
  repository(name: $name, owner: $owner) {
    ref(qualifiedName: $branch) {
      target {
        ... on Commit {
          history(first: 1) {
            nodes {
              oid
            }
          }
        }
      }
    }
  }
}
```

Variables `{owner, name, branch}`. Read: `repository.ref?.target.history.nodes[0].oid`. `ref` may be
`null` — the server falls back to the REST `getLatestCommit().uniqueId`.

### 2. `createCommitOnBranch` — `github.service.ts:1523`, `githubBypass.service.ts:1822`

```graphql
mutation CreateCommit($input: CreateCommitOnBranchInput!) {
  createCommitOnBranch(input: $input) {
    commit {
      url
      oid
    }
  }
}
```

Variables: `{input: {branch: {repositoryNameWithOwner, branchName}, message: {headline, body?},
fileChanges: {additions: [{path, contents}], deletions: [{path}]}, expectedHeadOid}}`.
Read: `createCommitOnBranch.commit.url` (→ `link`) and `.commit.oid` (→ `sha`).

Error `type` values branched on (`github.service.ts:1608-1655`):

| `errors[0].type` | Result |
|---|---|
| `NOT_FOUND` | 404; message containing `Could not resolve to a Repository` gets a custom summary |
| `BRANCH_PROTECTION_RULE_VIOLATION` | 403, `branchProtected: true`; message containing `Changes must be made through a pull request` gets a custom summary |
| `FORBIDDEN` | 403; `branchProtected: true` only when the message contains `Changes must be made through a pull request` |
| `STALE_DATA` | 412, "Your branch is not up to date. Please try again" |
| anything else | 500 |

Payloads are batched at `MAX_COMMIT_PAYLOAD_BYTES = 30 MiB`; each subsequent batch passes the prior
batch's returned `oid` as `expectedHeadOid` (`github.service.ts:1421,1455-1481`).

### 3. `getFileShaByPath` — `github.service.ts:873`, `githubBypass.service.ts:1169`

```graphql
query($owner: String!, $repo: String!, $expression: String!) {
  repository(owner: $owner, name: $repo) {
    object(expression: $expression) {
      ... on Blob {
        oid
      }
    }
  }
}
```

Variables `{owner, repo, expression: "<sha|ref>:<path>"}`. Read: `repository?.object?.oid ?? null`.
Both `repository` and `object` are nullable. All errors are swallowed to `null`.

### 4. `getBranches` — `github.service.ts:1022`, `githubBypass.service.ts:1315`

```graphql
query($owner: String!, $repo: String!, $cursor: String, $queryStr: String) {
  repository(owner: $owner, name: $repo) {
    refs(refPrefix: "refs/heads/", first: 100, after: $cursor, query: $queryStr) {
      pageInfo {
        hasNextPage
        endCursor
      }
      nodes {
        name
      }
    }
  }
}
```

Variables `{owner, repo, cursor: cursor || null, queryStr: q}`. Read:
`repository.refs.pageInfo.hasNextPage`, `.pageInfo.endCursor`, `.nodes[].name`. `getAllBranches`
loops until `hasNextPage` is false with **no page cap**.

### 5. `getRepos` — `github.service.ts:1146`, `githubBypass.service.ts:1439`

```graphql
query($owner: String!, $cursor: String) {
  repositoryOwner(login: $owner) {
    repositories(first: 100, after: $cursor, orderBy: { field: PUSHED_AT, direction: DESC }) {
      pageInfo {
        hasNextPage
        endCursor
      }
      nodes {
        name
      }
    }
  }
}
```

Variables `{owner, cursor}`. Read: `repositoryOwner?.repositories.pageInfo.{hasNextPage,endCursor}`
and `.nodes[].name`. A `null` `repositoryOwner` cleanly ends pagination.

### 6. `searchRepos` — `githubBypass.service.ts:1510` (bypass only)

```graphql
query($q: String!, $first: Int!) {
  search(query: $q, type: REPOSITORY, first: $first) {
    nodes {
      ... on Repository {
        nameWithOwner
      }
    }
  }
}
```

Variables `{q: "<sanitized> in:name user:<owner> fork:true", first: limit}` (default 50). Read:
`search.nodes[]?.nameWithOwner`, with `null` nodes filtered out.

### 7. `markPullRequestReadyForReview` — `github.service.ts:1743`, `githubBypass.service.ts:2024`

```graphql
mutation ($pullRequestId: ID!) {
  markPullRequestReadyForReview(input: { pullRequestId: $pullRequestId }) {
    pullRequest {
      id
    }
  }
}
```

Variables `{pullRequestId: pr.node_id}` from a preceding `GET /pulls/{n}`. Response unused.

### 8. `convertPullRequestToDraft` — `githubBypass.service.ts:2094` (bypass only)

```graphql
mutation ($pullRequestId: ID!) {
  convertPullRequestToDraft(input: { pullRequestId: $pullRequestId }) {
    pullRequest {
      id
    }
  }
}
```

Variables `{pullRequestId: pr.node_id}`. Response unused.

### 9. `addPullRequestReviewThread` — `githubBypass.service.ts:485` (bypass only)

```graphql
mutation ($pullRequestReviewId: ID!, $path: String!, $body: String!) {
  addPullRequestReviewThread(
    input: { pullRequestReviewId: $pullRequestReviewId, path: $path, body: $body, subjectType: FILE }
  ) {
    thread {
      id
    }
  }
}
```

Variables `{pullRequestReviewId: pendingReview.node_id, path, body}`. Read:
`addPullRequestReviewThread.thread.id` — a `null` thread throws
`GitHub returned no review thread for <path> on PR <n>` and the pending review is deleted.

### 10. `BlameAuthors` — `githubUpdate.service.ts:167`

```graphql
query BlameAuthors($owner: String!, $name: String!, $ref: String!, $path: String!) {
  repository(owner: $owner, name: $name) {
    ref(qualifiedName: $ref) {
      target {
        ... on Commit {
          blame(path: $path) {
            ranges {
              startingLine
              endingLine
              commit {
                author {
                  email
                  user {
                    email
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

Variables `{owner, name, ref: "refs/heads/<deployBranch>", path: <contentDirectory>/<filePath>}`.
Read: `repository.ref?.target.blame.ranges[]` (`ref` nullable → `[]`), each `.startingLine`,
`.endingLine`, `.commit.author?.user?.email` preferred over `.commit.author?.email` (trimmed,
lowercased). All errors swallowed.

### 11. `Blame` — `githubUpdate.service.ts:243`

```graphql
query Blame($owner: String!, $name: String!, $ref: String!, $path: String!) {
  repository(owner: $owner, name: $name) {
    ref(qualifiedName: $ref) {
      target {
        ... on Commit {
          blame(path: $path) {
            ranges {
              startingLine
              endingLine
              commit {
                committedDate
              }
            }
          }
        }
      }
    }
  }
}
```

Same variables. Read: `ranges[].startingLine`, `.endingLine`, `.commit.committedDate` (must parse
via `new Date(...)`). Unlike the authors variant this one **rethrows** on failure.

## GET /repos/{owner}/{repo}/git/ref/{ref}

Call sites: `github.service.ts:1306` (`getRef` → `checkIfRefExists`);
`onboardingTemplateSeed.service.ts:87` (`waitForDefaultBranchHeadSha`); `hostedRepo.service.ts:583,726`.

**Request:** `ref` as `heads/<branch>` (no `refs/` prefix).

**Response:**

```json
{ "ref": "refs/heads/main", "object": { "sha": "1111111111111111111111111111111111111111", "type": "commit" } }
```

`checkIfRefExists` reads truthiness only. The seeding paths read `object.sha`; an empty string yields
`"Target branch ref had no commit SHA"`.

**Error behavior.** `checkIfRefExists` swallows everything to `false`.
`waitForDefaultBranchHeadSha` retries `404` with backoff `[100, 250, 500, 1000, 2000] ms`; exhausting
those on `404` gives "Target repository has no commits on the default branch yet. Expected the repo
to be created with auto_init: true." Non-404 ⇒ "Failed to read target repository branch".

## POST /repos/{owner}/{repo}/git/refs

Call sites: `github.service.ts:1332`, `githubBypass.service.ts:1630`.

**Request:** `{ref: "refs/heads/<branch>", sha}`. **Response fields read: none** — a `201` with any
body suffices.

**Error behavior.** The bypass path checks `status === 422` **and**
`response.data.message === "Reference already exists"` by **exact string equality**
(`githubBypass.service.ts:2308-2321`) ⇒ "Cannot create branch with a duplicate name". Anything else
rethrows. The octokit path instead pre-checks existence via `GET /git/ref/{ref}`
(`github.service.ts:1971`).

## DELETE /repos/{owner}/{repo}/git/refs/{ref}

Call sites: `github.service.ts:2022`, `githubBypass.service.ts:2367`. Preceded by
`GET /repos/{owner}/{repo}` reading only `default_branch` (deleting the default branch is refused
client-side).

**Request:** `ref` as `heads/<branch>`. **Response: none read** — return `204`.

**Error behavior:** `403` "Insufficient permissions to delete this branch.", `404` "Branch not
found.", `409` "Branch cannot be deleted due to a conflict.", `422` "Branch is protected or does not
exist.", anything else logged and 500.

---

# Tier 3 — PR flow

## GET /repos/{owner}/{repo}/pulls/{pull_number}

**89k/7d.** The most field-hungry PR read; 15 call sites.

Call sites: `github.service.ts:407` (`getPullRequestSourceBranch`), `:1756`
(`markPrReadyForReview`), `:2255` (`getPullRequestSummary`); `githubBypass.service.ts:320`, `:337`
(`getPullRequestAuthor`), `:1909`, `:2074` (`convertPrToDraft`), `:2554`;
`api/agentApi/vercel/tools/readPrs.ts:236`;
`api/services/writerService/tools/fetchPullRequest.ts:69`;
`api/controllers/dashboard/workflow.controller.ts:1629`;
`api/utils/git.ts:237` (raw `fetch`, not octokit).

**Response — union of every read field:**

```json
{
  "number": 412,
  "node_id": "PR_kwDOABCD1M5abcde",
  "title": "Docs: add quickstart",
  "body": "Adds a quickstart page.",
  "state": "open",
  "draft": false,
  "merged": false,
  "merged_at": null,
  "merge_commit_sha": null,
  "created_at": "2026-08-30T09:00:00Z",
  "updated_at": "2026-08-30T10:16:00Z",
  "closed_at": null,
  "html_url": "https://github.com/acme/docs/pull/412",
  "commits": 3,
  "additions": 120,
  "deletions": 4,
  "changed_files": 2,
  "head": { "ref": "editor/quickstart", "sha": "1111111111111111111111111111111111111111", "repo": { "id": 1234567 } },
  "base": { "ref": "main", "repo": { "id": 1234567 } },
  "user": { "id": 987, "login": "ada", "type": "User", "avatar_url": "https://avatars.githubusercontent.com/u/987" }
}
```

Who reads what:
- `head.ref` — `getPullRequestSourceBranch` (`:416`), `readPrs`, the raw `fetch` in `git.ts:250`.
- `merged`, `state`, `draft`, `node_id`, `html_url` — `markPrReadyForReview` / `convertPrToDraft`
  (`:1765-1800`). `merged: true` ⇒ 409 refusal; `state === "closed"` ⇒ a reopen PATCH first;
  `draft: true` ⇒ the GraphQL mutation.
- `user.login`, `user.type` — `getPullRequestAuthor` (`githubBypass.service.ts:345`).
- Everything in `mapGithubPullRequest` (below) — `getPullRequestSummary`.
- `merge_commit_sha`, `merged_at` — `readPrs.ts:246-247`.
- `commits`, `additions`, `deletions`, `changed_files` — `fetchPullRequest.ts:84-87`.

**`mapGithubPullRequest` (`api/utils/git.ts:99-113`)** reads `number`, `title`, `html_url`,
`head.ref`, `head.repo` (**nullable**) `.id`, `base.ref`, `base.repo.id` (non-null), `updated_at`,
`user` (nullable) `.id`/`.login`/`.avatar_url` (optional), plus `draft?`, `state`, `merged_at`. It
produces `{number, title, url, state, headRefName, baseRefName, isCrossRepository, updatedAt, author}`.

**`getGithubPullRequestStatusString` (`git.ts:115-128`)** — strict precedence:
`merged_at !== null → "merged"`; else `state !== "open" → "closed"`; else `draft === true → "draft"`;
else `"open"`. A merged PR reports `merged` regardless of `state`, and `draft` is only visible on
open PRs.

**Fidelity notes.** A fork PR has `head.repo === null`, which makes
`isCrossRepository = (undefined !== base.repo.id) = true` — the intended behavior. `404` ⇒
`getPullRequestSummary` returns `null`; other statuses rethrow.

## GET /repos/{owner}/{repo}/pulls

Call sites: `github.service.ts:1824` (`findOpenPrForRef`), `:2235` (`listOpenPullRequests`),
`:2274` (`getPullRequestStatus`); `githubBypass.service.ts` mirrors;
`workflow.controller.ts:1662`; `listPullRequests.ts:97,138`.

**Request params by call site:**

| Call site | Params |
|---|---|
| `findOpenPrForRef` | `head: "<owner>:<ref>"`, `state: "open"`, `per_page: 1` |
| `listOpenPullRequests` | `state: "open"`, `per_page: 100`, `sort: "updated"`, `direction: "desc"` |
| `getPullRequestStatus` | `head: "<owner>:<branch>"`, `state: "all"`, `per_page: 5`, `sort: "popularity"` |
| `workflow.controller` | `state: "all"`, `sort: "updated"`, `direction: "desc"`, `per_page` |
| writer tool | `state`, `sort: "created"`, `direction: "desc"`, `per_page` 20 or 100, `page` |

**Response:** an array of the PR object above. `findOpenPrForRef` reads `[0].html_url`, `[0].head.sha`,
`[0].number`. `getPullRequestStatus` sorts client-side by `{open:0, draft:1, merged:2, closed:3}` and
by `created_at` among open PRs, then reads `number`, `title`, `body`, `html_url`, `created_at`,
`merged_at`, `head.sha`.

**Fidelity note.** The `head: "<owner>:<branch>"` filter must actually filter — `getPullRequestStatus`
depends on it to find the PR for a specific branch, then cross-checks `head.sha` against
`GET /branches/{branch}`'s `commit.sha` to decide whether a merged/closed PR is stale.

## PUT /repos/{owner}/{repo}/pulls/{pull_number}/merge

**12k/7d.** `github.service.ts:432`, `githubBypass.service.ts:377`.

**Request:** `{merge_method}` — resolved from `GET /repos/{owner}/{repo}` when not supplied.
**Response: none read.** A successful merge is followed by a `GET /pulls/{n}` for `head.ref`, whose
failure is only warned about.

**Error behavior:** `405` "Pull request is not mergeable. It may have conflicts or require
reviews."; `409` "Head branch was modified. Review and try the merge again."; `404` "Pull request
not found."; anything else 500.

## POST /repos/{owner}/{repo}/pulls

`github.service.ts:1685`, `githubBypass.service.ts:1969`.

**Request:** `{title, body, base: <deployBranch>, head: <ref>, draft}`.
**Response read:** `data.html_url`, `data.head.sha`, `data.number`.

**Error behavior.** `422` whose body matches
`{"errors": [{"resource","code","message"}]}` with a `message` containing
`A pull request already exists` ⇒ the server falls back to `findOpenPrForRef` and returns that PR as
a success. A `message` containing `No commits between ` ⇒ "No files have been changed on this
branch." (`api/validators/git.validators.ts:66-99`).

## PATCH /repos/{owner}/{repo}/pulls/{pull_number}

`github.service.ts:511` (`title`), `:558` (`state: "closed"`), `:1777` (`state: "open"`);
`editorCommentPrSync.service.ts:321` (`body`). **Response: none read.** `404` ⇒ "Pull request not
found."; otherwise 500.

## GET /repos/{owner}/{repo}/pulls/{pull_number}/files

Call sites: `github.service.ts:250` (`listPullRequestFiles`); `githubBypass.service.ts:362`
(`listPrFilePaths`); `editorCommentPrSync.service.ts:373`; `fetchPullRequest.ts:90`;
`AutomationRun.ts:765`; `utils/github/getChangedFilePaths.ts:16` (`octokit.paginate`).

**Request:** `per_page: 100`, `page`. **Response — an array; read fields:**

```json
[
  { "filename": "docs/index.mdx", "status": "modified", "patch": "@@ -1,3 +1,4 @@\n..." }
]
```

`filename` everywhere; `status` in the comment-sync path; `patch` in `fetchPullRequest.ts:102`.

**Fidelity notes.** `listPullRequestFiles` pages manually up to **30 pages** and stops when a page
returns fewer than 100 items, marking `truncated: true` once `GITHUB_PR_FILES_CAP = 3000` is reached
(`github.service.ts:246-269`). `getChangedFilePaths` uses `octokit.paginate`, which needs a working
`Link` header. GitHub itself caps this endpoint at 3000 files.

## GET / POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews

**GET** — `github.service.ts:2313`, `githubBypass.service.ts:2658` (`per_page: 100`, single request,
no pagination); `readPrs.ts:154` (`octokit.paginate`, **Link header required**).

```json
[
  { "state": "APPROVED", "body": "LGTM", "submitted_at": "2026-08-30T11:00:00Z", "user": { "login": "ada" } }
]
```

`state` is switched over `CHANGES_REQUESTED` / `COMMENTED` / `APPROVED` with precedence
approved & changes_requested > commented, default `"waiting"`. `readPrs` additionally reads
`user.login` (fallback `"unknown"`), `submitted_at` (`?? null`), `body`.

**POST** — two shapes in `githubBypass.service.ts`:
1. `:502` — **pending review**: body is `{}` (only path params). Reads `data.id` and **`data.node_id`**,
   which is fed to the `addPullRequestReviewThread` GraphQL mutation. The fake must return a
   `PENDING` review with a usable GraphQL node id.
2. `:592` — direct submit: `{event: "APPROVE" | "REQUEST_CHANGES", body?}`. Response unused.

Follow-ups: `POST .../reviews/{review_id}/events` with `{event: "REQUEST_CHANGES", body}` (response
unused, `:535`) and `DELETE .../reviews/{review_id}` for cleanup on any failure (`:564`, errors only
warned).

**Error behavior** (`reviewErrorResult`, `githubBypass.service.ts:606`): `422` whose `errors[]`
contains (as a string or `{message}`) a case-insensitive `your own pull request` ⇒ "You cannot
<verb> your own pull request."; other `422` ⇒ "GitHub could not process this review…"; `403` ⇒
permission message; `404` ⇒ "Pull request not found."

## GET / POST /repos/{owner}/{repo}/issues/{issue_number}/comments and PATCH .../issues/comments/{comment_id}

`github.service.ts:324` (GET, `per_page: 100`), `:285`/`:355` (POST), `:342` (PATCH);
`readPrs.ts:111` (`octokit.paginate`); `fetchPullRequest.ts:115`; `AutomationRun.ts:979,986`;
`processCommentBotJob.ts:58`.

**GET response:**

```json
[
  { "id": 22334455, "body": "<!-- mintlify-marker -->\nPreview ready", "created_at": "2026-08-30T10:20:00Z", "user": { "login": "ada", "name": "Ada Lovelace" } }
]
```

`upsertPrComment` finds the first comment whose `body` includes a marker tag and reuses its `id`
(`github.service.ts:334-345`). `readPrs` reads `user.login`, `created_at`, `body`;
`fetchPullRequest` reads `user?.name` and `body`.

**POST response:** only `data.id` is read, and only by `AutomationRun.ts:997`.
**PATCH response:** unread.

## GET /repos/{owner}/{repo}/branches/{branch}

**14k/7d.** `github.service.ts:2153` (`getBranchProtections`), `:2357` (`getBranchHeadSha`);
`githubBypass.service.ts:2702`.

```json
{ "name": "main", "protected": true, "commit": { "sha": "1111111111111111111111111111111111111111" } }
```

Only `protected` and `commit.sha` are read. `protected: false` short-circuits
`getBranchProtections` to all-false without calling the rules endpoint. `getBranchHeadSha` swallows
every error to `undefined`.

## GET /repos/{owner}/{repo}/rules/branches/{branch}

**5.7k/7d.** Two independent consumers with different rule types.

Call sites: `github.service.ts:2174` and `githubBypass.service.ts:2521` (`getBranchProtections`);
`api/utils/github-commit-message-regex.ts:39` (`getCommitMessageRegex`).

**Response — a flat array of rules:**

```json
[
  { "type": "pull_request", "parameters": { "required_approving_review_count": 1, "require_code_owner_review": false } },
  { "type": "non_fast_forward" },
  { "type": "commit_message_pattern", "parameters": { "operator": "regex", "pattern": "^(feat|fix|docs): ", "negate": false } }
]
```

- `getBranchProtections` finds `type === "pull_request"` (reading
  `parameters.required_approving_review_count > 0` and `parameters.require_code_owner_review`) and
  tests for any `type === "non_fast_forward"` (→ `allowsForcePush: !hasNonFastForward`). **An empty
  array while `branch.protected` is true is interpreted as "legacy protection"** and yields
  `{enabled: true, requiresApprovals: true}` (`github.service.ts:2199-2207`).
- `extractRegexFromGithubBranchRules` (`github-commit-message-regex.ts:50-66`) returns the first rule
  where `type === "commit_message_pattern"` **and** `parameters` is an object **and**
  `parameters.negate !== true` **and** `parameters.operator === "regex"` **and**
  `parameters.pattern` is a non-blank string.

**Error behavior.** `getBranchProtections` maps `404` to all-false. `getCommitMessageRegex`
swallows **every** throw to `undefined`.

## GET /repos/{owner}/{repo}/commits/{commit_sha}/pulls

`github.service.ts:1846`, `githubBypass.service.ts:2175`. `commit_sha` is `this.sha ?? this.ref`
(so a branch name is also passed here).

```json
[
  { "closed_at": null, "created_at": "2026-08-30T09:00:00Z", "html_url": "https://github.com/acme/docs/pull/412", "head": { "ref": "editor/quickstart" }, "base": { "ref": "main" } }
]
```

`getBranchInfo` picks the first entry with `closed_at == null && base.ref === deployBranch &&
head.ref === this.ref`, reading `created_at` and `html_url`.

## GET /repos/{owner}/{repo}/collaborators/{username}/permission

`github.service.ts:2132`, `githubBypass.service.ts:2477`.

```json
{ "permission": "write" }
```

`permission === "write" || permission === "admin"` ⇒ write access. Every error is swallowed to
`false`. GitHub's legacy vocabulary (`admin`/`write`/`read`/`none`) is what's expected.

## POST /repos/{owner}/{repo}/merges

`githubBypass.service.ts:756` only (`mergeBranch`).

**Request:** `{base: <targetBranch>, head: <branch>, commit_message: "Merge <branch> into <target>"}`.
**Response read:** `data.html_url` only, typed optional and defaulted `response.data ?? {}` — so a
`204 No Content` ("nothing to merge") body must be tolerated; the link then falls back to a
synthesized tree URL.

**Error behavior:** `403` "Merge is not permitted…", `409` "Branch is not mergeable. It may have
conflicts.", `404` "Branch not found.", else 500.

## GET /repos/{owner}/{repo}/commits/{ref}/check-runs

`api/agentApi/vercel/tools/readPrs.ts:173` via `octokit.paginate` — **the envelope must be
`{total_count, check_runs: [...]}` with Link pagination.** Params: `ref: <headSha>`,
`filter: "latest"`, `per_page: 100`.

```json
{
  "total_count": 1,
  "check_runs": [
    { "name": "Mintlify Deployment", "status": "completed", "conclusion": "success", "output": { "summary": "Deployed", "text": null } }
  ]
}
```

Read: `name`, `status`, `conclusion` (`?? null`), `output.summary`, `output.text`. `output` may be
`null`. Runs are sorted failing-first; `success`/`skipped`/`neutral` count as passing and their
`output.text` is dropped.

## GET /repos/{owner}/{repo}/pulls/{pull_number}/comments and /commits

`readPrs.ts:117` (`sort: "created"`, `direction: "desc"`, `per_page: 50`) reads `user.login`,
`created_at`, `body`, `path`, `line`, `original_line`. `fetchPullRequest.ts:106` reads `user.name`
and `body`. `AutomationRun.ts:770,1117` calls `/pulls/{n}/commits` with `per_page: 100` and reads
only `author?.login`.

---

# Tier 4 — accept-and-forget stubs

## POST /repos/{owner}/{repo}/deployments and /deployments/{deployment_id}/statuses

**8k/7d.** `github.service.ts:1358` / `:1401`; `githubBypass.service.ts:1656` / `:1700`.

**Deployment request:** `{ref, description, environment, transient_environment, required_contexts: [],
auto_merge: false}` with header `X-GitHub-Api-Version: 2022-11-28`. `environment` is one of
`production` / `staging` / `staging - <contentDirectory>` / `qa` (`api/types/GithubDeployment.ts`).
`task`, `payload`, `production_environment` are never sent.

**Deployment response:** `{"id": 1234567890}` — the code guards `if (!('id' in data)) throw
Error('unable to create deployment')`, so the fake must **not** return the `{message: ...}` 202
auto-merge body. Only `id` is consumed.

**Status request:** `{state, log_url, description, environment, environment_url, auto_inactive}` with
the same API-version header. In practice only `state: "success"` is sent.
**Status response: no field is read.**

## GET /app/installations/{installation_id}

Note **both** `{installation_id}` and `{installationId}` path templates are used against the same
endpoint (`githubApp.service.ts:262` vs `:327`) — a fake must accept both literal strings or
normalize.

```json
{
  "id": 55555555,
  "account": { "login": "acme", "type": "Organization" },
  "repository_selection": "all",
  "suspended_by": null,
  "permissions": { "contents": "write", "pull_requests": "write", "checks": "write", "metadata": "read" }
}
```

- `account.type === "User"` ⇒ `"user"`, else `"org"`; `account.login` is lowercased in the ownership
  path and requires `'login' in data.account`.
- `suspended_by` truthy ⇒ `AccessTypeV2.SUSPENDED` (short-circuits the repository listing).
- `repository_selection` must be exactly `"selected"` or `"all"` — anything else throws
  `"Invalid repository selection"`.
- `permissions` is returned wholesale.

`404` (duck-typed via `'status' in err`) ⇒ `null`; other statuses rethrow.
`DELETE /app/installations/{id}` treats `404` as already-deleted success.

## GET /installation/repositories

Two consumers with different shapes:
- `githubApp.service.ts:277` — `paginate.iterator(..., {per_page: 30})` with `for await`, reading
  `repo.name`. **Requires working Link-header pagination.**
- `slackAgentRunSupport.ts:287` — a single request reading `data.repositories[].full_name` and
  **`data.total_count`**, with an `AbortController` signal.

```json
{ "total_count": 2, "repositories": [ { "name": "docs", "full_name": "acme/docs" } ] }
```

## GET /user/installations

`githubApp.service.ts:321` — `userOctokit.paginate('GET /user/installations')` with no params.
Octokit unwraps the `installations` array, so the envelope must be
`{"total_count": N, "installations": [{"id": 55555555}]}`. Only `installation.id` is read; a miss
throws `does not have access to installation with id <n>`.

## GET /user and GET /users/{username}

`GET /user` fields read: `data.id` and `data.login` (lowercased) at `githubApp.service.ts:231-236`;
`data.login` at `hostedRepo.service.ts:631`, `dashboard.service.ts:191,580`,
`onboarding.controller.ts:274`. `githubApp.service.ts:65` uses it as a bare validity probe.

**`onboarding-github-repo.service.ts:157-161` reads no body at all** — only the
**`x-oauth-scopes` response header**, comma-split and trimmed; containing `repo` ⇒ can access
private repos. This header is load-bearing.

`GET /users/{username}` (`gitUserSlackMapping.service.ts:29`) reads only `data.name` (trimmed,
nullable) and `data.email` (nullable); every error becomes `undefined`.

Adjacent: `GET /user/orgs` with `per_page: 100` (no pagination loop) reads `org.login`
(`onboarding.controller.ts:293`).

## GET /user/repos and repo creation

`GET /user/repos` (`onboarding-github-repo.service.ts:164`): params `page`, `per_page: 100`,
`sort: "updated"`, `affiliation: "owner,collaborator,organization_member"`. Reads per element:
`default_branch`, `full_name`, `private`, `name`, `owner.login`. Manual page loop, stops when a page
returns fewer than 100 or after 500 repos. Does **not** use `octokit.paginate` or the Link header.

`POST /user/repos` and `POST /orgs/{org}/repos`: body `{name, private, auto_init: true}`; reads
`data.name` and `data.default_branch`. `422` ⇒ try the next name candidate; `403` ⇒
`reason: "forbidden"`.

`PATCH /repos/{owner}/{repo}` (`hostedRepo.service.ts:328,406,493,510`) sends `is_template: true` or
`private: false`; responses ignored. `DELETE /repos/{owner}/{repo}` is best-effort cleanup with
errors swallowed.

`GET /orgs/{org}/repos` (`hostedRepo.service.ts:441`) with `per_page: 1`, `type: "all"` exists only
to **read the `Link` header** — `parseLastPageFromLinkHeader` (`utils/hostedRepoOrg.ts:26-38`) finds
`rel="last"` and extracts its `page=` query param as the repo count, falling back to `data.length`.
The Link header format is load-bearing here.

## POST /repos/{template_owner}/{template_repo}/generate

`onboardingTemplateSeed.service.ts:726`.

**Request:** `{owner, name, private}` plus the two path params. No `description`, no
`include_all_branches`, no custom headers.
**Response read:** `data.name`, `data.default_branch`.

**Error branching** (all require `err.response.data.message`):

| Status + message | Result |
|---|---|
| `422` + `/template repository/i` or `/is not a template/i` | fall back to tarball seeding |
| `422` + `/repository limit\|repository_limit\|maximum number of repositories/i` | hard error, raw message surfaced |
| `422` otherwise (name taken) | try the next name candidate; exhausting them ⇒ `all_taken` |
| `404` | fall back to tarball seeding |
| `403` + repo-limit message | hard error |
| `403` otherwise | `forbidden` |

**Fidelity note.** The generate path is skipped entirely when the template source has a non-empty
subpath — those always take the tarball/git-tree route. The seed commit message must be
`"Initial commit"` (`HOSTED_ONBOARDING_INITIAL_COMMIT_MESSAGE`) because the push webhook dedupes on
it.

## POST git/blobs, POST git/trees, POST git/commits, PATCH git/refs/{ref}

The template-seeding and repo-copy write chain (`onboardingTemplateSeed.service.ts:499-521,662`;
`hostedRepo.service.ts:702-751`).

- `POST /repos/{owner}/{repo}/git/blobs` — `{content: <base64>, encoding: "base64"}`; reads
  `data.sha` only. Uploaded by 8 concurrent workers.
- `POST /repos/{owner}/{repo}/git/trees` — `{tree: [{path, mode: "100644"|"100755", type: "blob",
  content}| {path, mode, type: "blob", sha}]}`. **No `base_tree`.** Reads `data.sha`. The code relies
  on `content` being interpreted as UTF-8 and any per-item `encoding` being ignored
  (`onboardingTemplateSeed.service.ts:562-565`).
- `POST /repos/{owner}/{repo}/git/commits` — `{message, tree, parents: [sha]}`; no `author`/
  `committer`. Reads `data.sha`.
- `PATCH /repos/{owner}/{repo}/git/refs/{ref}` — `ref: "heads/<branch>"`, `{sha}`. No `force`.
  Response ignored.
- `GET /repos/{owner}/{repo}/git/commits/{commit_sha}` (`hostedRepo.service.ts:592`) reads
  `data.tree.sha`.

No status branching on any of these — failures collapse into
`"Failed to finalize template commit on target repository"` plus a best-effort repo delete.

## GET /repos/{owner}/{repo}/readme

Two representations are both required:
- `repoDescription.service.ts:43` sends `mediaType: {format: "raw"}` and consumes `String(r.data)` —
  **raw text, no JSON**. Errors swallowed to `""`.
- `discoverProjectContext.ts:108,200` uses `repos.getReadme` (default JSON) and reads `data.content`,
  base64-decoding it; the second call also accepts an optional `ref`. Errors swallowed.

## GET /repos/{owner}/{repo}/zipball/{ref} and /tarball/{ref}

- **zipball** (`hostedRepo.service.ts:536`): preceded by `GET /repos/{owner}/{repo}` for
  `default_branch`, which becomes `ref`. `response.data` must be an `ArrayBuffer` or `Buffer` —
  `deployment.controller.ts:4695` 500s with `"Unexpected response format from GitHub"` otherwise.
- **tarball** (`onboardingTemplateSeed.service.ts:228`): `ref: ""` (empty string ⇒ default branch)
  with `request: {redirect: "follow"}` — **the fake must serve a 302 to a downloadable gzip archive**.
  `response.data` is type-checked as `ArrayBuffer`; anything else ⇒ `"Unexpected tarball response
  type from GitHub"`. The extractor strips the first path segment (GitHub's `<owner>-<repo>-<sha>/`
  prefix) and uses tar `header.name`, `header.type` (only `file`/`contiguous-file`), and the
  executable bit of `header.mode`. Archives over 50 MiB (compressed or cumulative uncompressed) are
  rejected.

## GET /search/issues (`search.issuesAndPullRequests`)

`workflow.controller.ts:1648`. Query: `q: "repo:<owner>/<repo> is:pr <search>"`, `per_page`. No
`page`, `sort`, `order`, or `advanced_search`.

```json
{
  "items": [
    { "number": 412, "title": "Docs: add quickstart", "state": "open", "html_url": "https://github.com/acme/docs/pull/412", "updated_at": "2026-08-30T10:16:00Z", "pull_request": { "merged_at": null }, "user": { "login": "ada" } }
  ]
}
```

Read: `items[].number`, `.title`, `.pull_request?.merged_at`, `.state`, `.html_url`, `.updated_at`,
`.user?.login`. **`total_count` and `incomplete_results` are not read.**

## Reactions

`processCommentBotJob.ts:59,68,87,96` and `processGhCommentPrAgentJob.ts:88,97,116,125`:

- `POST /repos/{o}/{r}/pulls/comments/{comment_id}/reactions` — `{content: "+1"}`
- `POST /repos/{o}/{r}/issues/comments/{comment_id}/reactions` — `{content: "+1"}`
- `DELETE /repos/{o}/{r}/pulls/comments/{comment_id}/reactions/{reaction_id}`
- `DELETE /repos/{o}/{r}/issues/comments/{comment_id}/reactions/{reaction_id}`

Response fields read at these four sites: none. The one that matters is the earlier `eyes` reaction
create, whose `data.id` is persisted as `run.eyesReactionId` and later used as `{reaction_id}`. All
calls are wrapped in warn-only try/catch with no status branching.

## POST /repos/{owner}/{repo}/pulls/{pull_number}/requested_reviewers

`AutomationRun.ts:1038` — `{reviewers: [login]}`, one reviewer per call, iterating until one
succeeds. Response unread.

---

# Tier 5 — Webhooks

## Transport and signature

**Mounts** (`entryPoints/server.ts`):
- `:376` — `app.use('/github-webhook', logger, loggerMiddleware, githubWebhooksRouter)` — github.com.
- `:407` — `app.use('/github-enterprise/:subdomain', ...)`; an unregistered subdomain gets **404 with
  an empty body**.

**No body parser is installed.** There is no `express.json()` / `express.raw()` before the handler —
octokit's `createNodeMiddleware` (`githubWebhooks.router.provider.ts:92`, `path: '/'`) reads the raw
request stream itself and concatenates chunks as UTF-8. The fake must send a real request body, not
a pre-parsed one.

**Headers read** (all three required):

| Header | Use |
|---|---|
| `x-github-event` | event name |
| `x-hub-signature-256` | `sha256=<lowercase hex>` HMAC-SHA256 over the raw body |
| `x-github-delivery` | delivery id (becomes `deliveryId` in the SQS envelope) |
| `content-type` | must start with `application/json` |

**Not read:** `x-github-hook-installation-target-id`, `x-github-hook-installation-target-type`,
`x-github-hook-id`, `user-agent`, `x-hub-signature` (sha1).

Secret: `env.WEBHOOK_SECRET` for github.com; the per-enterprise decrypted `webhookSecret` for GHES
(`utils/github/enterpriseRouters.ts:46-49`). Verification is `@octokit/webhooks-methods` — WebCrypto
`crypto.subtle.verify`, constant time. There is **no hand-rolled HMAC** for GitHub in this repo.

**Response codes emitted:**

| Condition | Status | Body |
|---|---|---|
| Unparseable request URL | `422` | `{"error":"Request URL could not be parsed: <url>"}` |
| Method ≠ POST | `404` | `{"error":"Unknown route: <METHOD> <url>"}` |
| Missing/non-JSON `content-type` | `415` | `{"error":"Unsupported \"Content-Type\" header value..."}` + `accept: application/json` |
| Any required header missing | `400` | `{"error":"Required headers missing: <list>"}` |
| **Signature mismatch** | **`400`** | `{"error":"Error: [@octokit/webhooks] signature does not match event payload and secret"}` |
| Invalid JSON | `400` | `{"error":"Error: Invalid JSON"}` |
| Handler still running at 9000 ms | `202` | `still processing\n` |
| Success | `200` | `ok\n` |
| Handler throw | `500` | `{"error":"<Name>: <message>"}` |

**Fidelity note.** Handlers are awaited before the `200`, but in practice the delivery is published
to SQS first (`withQueueProduce` → `queueWebhook`, `githubWebhooks.router.provider.ts:111-164`) and
`ackWebhookReceived()` is a no-op, so the response is near-immediate. If SQS publish returns `false`
— which happens when no owner can be derived from
`repository.owner.login → installation.account.login → organization.login → sender.login` — the
handler runs inline instead. SQS send *errors* propagate and yield a `500`. Deliveries over the
256 KB SQS cap offload the payload to Redis under a `git-webhook-payload:` key.

## Subscribed events

Registered identically in `githubWebhooks.router.provider.ts:167-332` and
`api/workers/sqs/gitWebhookConsumer.ts:177-207`.

`installation.created`, `installation.deleted`, `github_app_authorization.revoked`,
`repository.renamed`, `repository.transferred`, `repository.privatized`, `repository.publicized`,
`installation_target.renamed`, `push`, `pull_request.closed`, `pull_request.opened`,
`pull_request.reopened`, `pull_request.edited`, `pull_request.ready_for_review`,
`pull_request.converted_to_draft`, `pull_request.synchronize`, `create`, `delete`,
`issue_comment.created`, `pull_request_review_comment.created`, `pull_request_review.submitted`.

**Explicitly not handled:** `ping`, `check_run` (including `rerequested` and `requested_action`),
`check_suite`, `installation_repositories` (`added`/`removed`), `installation.suspend`/`unsuspend`,
`member`, `membership`, `organization`, `team`, `status`, `workflow_run`, `workflow_job`, `release`,
`issues`, `star`, `fork`, `repository.created`/`.deleted`/`.archived`,
`pull_request.assigned`/`.labeled`/`.review_requested`, `issue_comment.edited`/`.deleted`,
`pull_request_review.edited`/`.dismissed`. Check runs exist only as *outbound* REST calls. An
unmapped event reaching the SQS consumer logs `'No handler for webhook event'` and is dropped
without error.

There is **no zod schema for the GitHub payload itself** — typing comes from
`EmitterWebhookEvent<'...'>`. The only runtime schema is the SQS envelope
(`api/validators/gitWebhookDelivery.validators.ts:5-14`):
`{deliveryId, event, createdAt (ISO datetime), provider: "github"|"github-enterprise", subdomain?,
dryRun, payload: unknown, payloadRef?}`.

## push

Handler `githubWebhooks.controller.ts:411` → `processPush:443` → `runOnPush:487`.

```json
{
  "ref": "refs/heads/main",
  "before": "0000000000000000000000000000000000000000",
  "after": "1111111111111111111111111111111111111111",
  "forced": false,
  "size": 2,
  "repository": { "name": "docs", "owner": { "login": "acme" }, "private": false, "default_branch": "main" },
  "installation": { "id": 55555555 },
  "pusher": { "name": "ada", "email": "ada@acme.com" },
  "sender": { "login": "ada", "type": "User" },
  "head_commit": {
    "id": "1111111111111111111111111111111111111111",
    "message": "docs: add quickstart",
    "timestamp": "2026-08-30T10:15:00Z",
    "tree_id": "4b825dc642cb6eb9a060e54bf8d69288fbee4904",
    "url": "https://github.com/acme/docs/commit/1111111111111111111111111111111111111111",
    "distinct": true,
    "author":    { "name": "Ada Lovelace", "email": "ada@acme.com", "username": "ada" },
    "committer": { "name": "Ada Lovelace", "email": "ada@acme.com", "username": "ada" },
    "added": ["docs/quickstart.mdx"],
    "modified": ["docs/docs.json"],
    "removed": []
  },
  "commits": [ "…same shape as head_commit…" ]
}
```

Field notes:
- `ref` must start with `refs/heads/`, else the push is ignored (`:515`).
- `before` — skipped when falsy or all-zeros; disables the net diff (`:541,549,1914`).
- `after` — compared against the last compare commit sha (`:1923`).
- `forced: true` forces a global update and disables net diff (`:542,550,697`).
- `head_commit` is required; `null` ⇒ skip. Its `message` is regex-matched for
  `/Merge pull request #(\d+)/` to derive a trigger PR number (`:1744`); its `author.name` plus
  `sender.login` drive the Mintlify-bot short-circuit (`:1708`); its `id` becomes
  `triggerGitSource.sha` (`:1754`).
- `installation.id` is required.
- `commits[]` is assumed **chronological** by both `aggregateCommits` and `attributeCommits`.
  `commits[].id` keys check-run creation and status; `.added`/`.modified`/`.removed` drive path
  aggregation; `.author.username`/`.author.name` seed git contributors; `.message` is tested for the
  editor trailer.
- `size` — the SQS consumer treats `commits.length < size` as an incomplete commit list
  (`gitWebhookConsumer.ts:102`), which is one of the conditions that keeps a push from being
  pre-filtered away.

**Fidelity notes.** `aggregateCommits` (`utils/github/aggregateCommits.ts`) folds the per-commit
path arrays into net `modified`/`removed` sets, then removes any `removed` path whose markdown base
name reappears in `added` (rename detection). `attributeCommits` marks each commit `noop` /
`tracking` / `outdated` relative to the content directory — a check run is created per commit, so a
push of N commits produces N `POST /check-runs`.

## pull_request.closed

Handler `:1274`, plus `syncEditorBranchFromPullRequest:1215` and `triggerMergeWorkflowsForPr:1764`.

Reads: `repository.owner.login`, `repository.name`, `repository.default_branch`; `installation`
(truthiness gate) and `installation.id`; `pull_request.number`, `.head.ref`, `.head.sha`,
`.head.repo.owner.login`, `.head.repo.name`, `.head.repo.id`, `.base.ref`, `.base.repo.id`,
`.merged`, `.state` (`=== "closed"`), `.closed_at`, `.html_url`, `.merge_commit_sha`,
`.user.login` / `.user.id` / `.user.avatar_url`, `.title`, `.updated_at`, `.draft`, `.merged_at`,
and (PostHog only) `.commits`, `.comments`, `.review_comments`, `.additions`, `.deletions`,
`.changed_files`.

## pull_request.opened

Handler `:2346`. Reads `repository.owner.login`, `repository.name`, `repository.private`;
`installation.id` (missing ⇒ returns after contributor seeding); `pull_request.number`,
`.base.ref`, `.head.ref`, `.head.sha`, `.head.repo.owner.login`, `.head.repo.name` (cross-repo PRs
are skipped), `.user.login`, `.user.name`, `.user.avatar_url`, plus everything
`mapGithubPullRequest` needs (`title`, `html_url`, `state`, `merged_at`, `updated_at`, `draft`,
`head.repo.id`, `base.repo.id`, `user.id`).

## pull_request.synchronize

Handler `:2421`. Reads `repository.owner.login`, `repository.name`, `repository.private`;
`installation.id` (required); `pull_request.base.ref`, `.base.sha`, `.head.ref`, `.head.sha`,
`.head.repo.owner.login`, `.head.repo.name`, `.user.login`, `.user.name`, `.user.avatar_url`.

**Fidelity note.** Unlike the other handlers this one dereferences `pull_request.head.repo`
**without** an optional chain (`:2439-2440`), so `head.repo` must be non-null on synchronize. It also
does not call `syncEditorBranchFromPullRequest`.

## pull_request.reopened / .edited / .ready_for_review / .converted_to_draft

Handler `:1242`. Reads `repository.owner.login`, `repository.name`; `pull_request.head.ref`,
`.base.ref`, `.head.repo.owner.login`, `.head.repo.name`; **`payload.action` is string-compared
`!== "reopened"`** at `:1258` (contributor seeding runs only on reopened + same-repo); plus the
`mapGithubPullRequest` fields.

## create and delete

Handlers `:2546` / `:2577`. Both require `ref_type === "branch"` (tags are ignored) and read
`repository.owner.login`, `repository.name`, `ref` (a **bare branch name**, not a `refs/heads/`
path), and `installation.id`.

## issue_comment.created

Handler `:2808`. Reads `comment.body` — must match `/@mintlify\b(?!\/)/` — `comment.id`,
`comment.user.type` (`=== "Bot"` ⇒ return), `comment.user.login` (fallback `"unknown"`);
`installation.id`; `repository.owner.login`, `repository.name`; `issue.number`; and
**`issue.pull_request` must be truthy** — plain issue comments are ignored.

## pull_request_review_comment.created

Handler `:2850`. Reads `comment.body`, `.id`, `.path`, `.line`, `.start_line`, `.user.type`,
`.user.login`; `installation.id`; `repository.owner.login`, `repository.name`;
`pull_request.head.ref`, `pull_request.number`.

## pull_request_review.submitted

Handler `:2884`. Reads `review.body` (nullable; falsy ⇒ return after emitting a socket event),
`review.id`, `review.user.type` (`=== "Bot"` ⇒ return), `review.user.login` (fallback `"unknown"`);
`installation.id`; `repository.owner.login`, `repository.name`; `pull_request.head.ref`,
`pull_request.number`.

## installation.* / repository.* / installation_target.renamed

| Event | Fields read |
|---|---|
| `installation.created` (`:302`) | `repositories` (may be absent ⇒ early return), `repositories[].full_name` (lowercased, split on `/`) |
| `installation.deleted` (`:329`) | `installation.id` |
| `github_app_authorization.revoked` (`:333`) | `sender.id` |
| `repository.renamed` (`:340`) | `changes.repository.name.from`, `repository.name`, `repository.owner.login` |
| `repository.transferred` (`:348`) | `changes.owner.from.user.login` (undefined ⇒ log and return), `repository.owner.login`, `repository.name` |
| `repository.privatized` (`:365`) / `.publicized` (`:389`) | `repository.owner.login`, `repository.name` |
| `installation_target.renamed` (`:396`) | `changes.login.from`, `account.login` |

## Consumer-side push pre-filter

`gitWebhookConsumer.ts:51-151` drops a push before it reaches the controller unless one of:
an active workflow schema push trigger matches `owner/repo` + branch (or
`repository.default_branch`); a deployment git source matches by content-directory overlap with the
changed paths, or by `deployBranch === branch`; or `commits.length < size` with at least one
deployment. A fake that wants pushes to actually deploy must satisfy one of these.

Replay-ready fixtures live at `test/controllers/githubWebhooks.controller.test.ts:186-334` and
`test/mocks/commitMocks.ts:3-21`.
