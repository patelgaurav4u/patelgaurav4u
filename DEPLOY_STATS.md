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
- Expiration: set a long window, or "No expiration"
- Repository access: **Public repositories (read-only)** is enough for public stats.
  To have private repos counted, choose **All repositories** instead.
- Permissions: no extra scopes are required for public-only stats.
- Copy the token (starts with `github_pat_`). You cannot view it again later.

> A classic token also works — create one with **no scopes ticked** for public data,
> or the `repo` scope if you want private repos counted.

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

## Notes

- `count_private=true` only counts private repositories when the *deploying* account's
  token has access to them — i.e. it works on your own instance, not on the shared one.
- `cache_seconds=86400` in the URLs asks for a longer cache, which reduces API calls
  and makes rate-limiting far less likely.
- The streak card (`streak-stats.demolab.com`) is a **different** project
  ([github-readme-streak-stats](https://github.com/DenverCoder1/github-readme-streak-stats))
  with its own healthy hosting, which is why it keeps working while the others fail.
  If you ever want to self-host it too, deploy that repo separately.
- GitHub proxies README images through Camo and caches them. After switching hosts,
  a card may show the old error for a while; a hard refresh or waiting out the cache
  clears it.
