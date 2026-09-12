# Deploying your own github-readme-stats instance

The GitHub stats cards in `README.md` are rendered by
[github-readme-stats](https://github.com/anuraghazra/github-readme-stats).

## Why the cards break

The project's shared public instance (`github-readme-stats.vercel.app`) is used by
hundreds of thousands of profiles. It regularly exhausts its GitHub API quota, and
when that happens every card it serves renders as:

```
Something went wrong! file an issue at https://tiny.one/readme-stats
No GitHub API tokens found
Please add an env variable called PAT_1 with your GitHub API token in vercel
```

This is not a problem with your README — the URL is correct, the upstream server is
simply out of tokens/quota. The only durable fix is to run your own instance with
your own GitHub token, which gets you a private 5,000 req/hr quota.

## One-time setup (~5 minutes)

### 1. Create a GitHub token

- Go to <https://github.com/settings/tokens?type=beta>
- **Generate new token** → fine-grained
- Expiration: prefer a fixed window (90 days / 1 year) over "No expiration", so a
  leaked token eventually dies on its own. You will need to rotate it in Vercel.
- Repository access: **Public repositories (read-only)** — see the scope note below.
- Permissions: no extra scopes are required for public-only stats.
- Copy the token (starts with `github_pat_`). You cannot view it again later.

> A classic token also works — create one with **no scopes ticked** for public data.
> Avoid the `repo` scope: it is coarse and grants full read/write to every private
> repository you can reach, which is far more than a stats card needs.

#### Choosing the repository scope

This token is stored as an environment variable in your Vercel project, and that
project runs a fork of third-party code. Anyone who can reach the Vercel project can
read the token. So scope it to the least thing that does the job.

| Scope | Cards work? | If the token leaks |
|---|---|---|
| **Public repositories (read-only)** — recommended | Yes | Grants nothing beyond what any visitor to your profile can already see |
| Only select repositories (a few of your own private repos) | Yes, plus those repos' commits | Exposes only the repos you hand-picked |
| All repositories | Yes, plus all private commits | Exposes **every private repo your account can read**, including employer/org repos |

**Recommendation: use Public repositories (read-only) and drop `count_private=true`
from the README URLs.** The only thing private access buys you is a larger number on
a decorative card; the cost is a credential that can read your private source code.

If the private commit count genuinely matters, use **Only select repositories** and
pick a handful of your *own* personal repos. Never include repositories owned by an
employer or a client — granting a third-party deployment read access to those is
likely a violation of your access agreement, regardless of intent.

### 2. Deploy to Vercel

- Fork <https://github.com/anuraghazra/github-readme-stats>
- Go to <https://vercel.com/new>, import your fork
- Framework preset: **Other**. Leave build settings at their defaults.
- Before clicking Deploy, expand **Environment Variables** and add:
  - Name: `PAT_1`
  - Value: the token from step 1
- Deploy. You will get a URL such as `https://your-project.vercel.app`

> The env var **must** be named `PAT_1` — that exact name is what the error message
> refers to. You can add `PAT_2`, `PAT_3`, … later to rotate across more tokens.

### 3. Point the README at your instance

Replace the host in both image URLs in `README.md`:

```
https://github-readme-stats.vercel.app  ->  https://your-project.vercel.app
```

Both the `/api?...` (stats) and `/api/top-langs/?...` (languages) URLs need it.

## Troubleshooting

Fetch the `vercel.app` URL directly with `curl` rather than judging from the README —
GitHub's Camo proxy caches card images and will keep serving a stale error:

```sh
curl -s "https://github-readme-stats-gaurav-six.vercel.app/api?username=patelgaurav4u" \
  | grep -o "Something went wrong\|[A-Z][a-z].*token[^<]*\|Total Stars"
```

The card prints the real cause on its **second line**. The two that matter:

### "No GitHub API tokens found"

No `PAT_1` env var reached the running deployment. Either it was never set, or it was
added after the last build — Vercel only picks up env vars at deploy time, so
**redeploy** after adding or changing one.

### "Resource not accessible by personal access token"

The token is being read, but lacks a permission the card needs. Typical symptom: the
**top-langs card works while the stats card fails**, because top-langs only lists
public repos, whereas the stats card also aggregates commits/PRs/issues over GraphQL.

Confirm it is the token and not your account by requesting any other username — if
that fails too, the token is the problem:

```sh
curl -s "https://github-readme-stats-gaurav-six.vercel.app/api?username=anuraghazra" \
  | grep -o "Resource not accessible[^<]*\|Total Stars"
```

Two fixes, in order of preference:

1. **Use a classic token with no scopes ticked** (<https://github.com/settings/tokens/new>).
   github-readme-stats is built around classic tokens and its GraphQL queries work
   reliably with them. An unscoped classic token reads only public data, so this stays
   least-privilege. Replace `PAT_1` in Vercel, then redeploy.
2. **Keep the fine-grained token** and add read access to your profile/user data under
   **Account permissions** (not the repository block). Fine-grained tokens have known
   friction with this project's account-level GraphQL calls.

## Notes

- `count_private=true` only counts private repositories when the *deploying* account's
  token has access to them. It has been removed from the README URLs, since it is a
  no-op with the recommended public-only token.
- Never paste the token into a chat, an issue, a commit, or a build log. It only ever
  needs to travel from GitHub to your Vercel project's environment variables. If it is
  exposed anywhere else, revoke it at <https://github.com/settings/tokens> and issue a
  new one — revoking is instant and free.
- `cache_seconds=86400` in the URLs asks for a longer cache, which reduces API calls
  and makes rate-limiting far less likely.
- The streak card (`streak-stats.demolab.com`) is a **different** project
  ([github-readme-streak-stats](https://github.com/DenverCoder1/github-readme-streak-stats))
  with its own healthy hosting, which is why it keeps working while the others fail.
  If you ever want to self-host it too, deploy that repo separately.
- GitHub proxies README images through Camo and caches them. After switching hosts,
  a card may show the old error for a while; a hard refresh or waiting out the cache
  clears it.
