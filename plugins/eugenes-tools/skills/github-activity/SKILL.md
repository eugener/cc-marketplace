---
description: GitHub activity dashboard - stars, forks, clones, views, issues, and PRs from other people across your repos. Use this whenever the user asks about GitHub activity, repo popularity, who starred or forked, traffic, external contributions, community engagement, what's happening on their repos, whether anyone is using their projects, repo stats, or GitHub notifications. Also trigger when the user mentions checking on their open source projects or wants a summary of external interest.
user-invocable: true
argument-hint: "[username-or-org]"
allowed-tools:
  - Bash
  - AskUserQuestion
---

Show a dashboard of external activity across a GitHub user's or organization's repositories. The goal is to answer "what are other people doing with my repos?" Uses the `gh` CLI exclusively.

## Step 0: Verify Prerequisites

```
gh auth status
```

If not authenticated, tell the user to run `gh auth login` and stop.

## Step 1: Resolve Target

If `$ARGUMENTS` is provided, use it as the GitHub login.

Otherwise, detect from the authenticated user:
```
gh api user -q '.login'
```

If that fails, ask the user via AskUserQuestion.

Determine whether the target is a user or organization:
```
gh api "users/<login>" -q '.type'
```
This returns `User` or `Organization`. Use the matching GraphQL root field in Step 2.

## Step 2: Fetch Repos with Stargazers and Forks (GraphQL)

A single GraphQL query replaces dozens of REST calls by fetching all repos with their stars, forks, issues, recent stargazers, and recent forks in one request. This matters because the REST alternative would need separate paginated calls per repo for stargazers and forks, easily hitting rate limits for users with many repos.

For a **user** (swap `user` for `organization` and `OWNER` for `NONE` if org):
```
gh api graphql --paginate -f query='
query($login: String!, $endCursor: String) {
  user(login: $login) {
    repositories(first: 100, after: $endCursor, ownerAffiliations: OWNER, isFork: false, orderBy: {field: STARGAZERS, direction: DESC}) {
      pageInfo { hasNextPage endCursor }
      nodes {
        nameWithOwner
        isArchived
        stargazerCount
        forkCount
        issues(states: OPEN) { totalCount }
        pushedAt
        stargazers(last: 10) {
          edges { starredAt node { login } }
        }
        forks(first: 10, orderBy: {field: CREATED_AT, direction: DESC}) {
          nodes { createdAt owner { login } }
        }
      }
    }
  }
}' -f login='<username>'
```

**Pagination caveat**: `gh api graphql --paginate` concatenates multiple JSON objects without a separator (e.g., `}{`). Split on `}\n{` or `}{` before parsing.

Parse the JSON response. Filter out archived repos. From this single response, extract:
- Repo list sorted by stars descending (cap display at top 20)
- 10 most recent stargazers per repo (with timestamps)
- 10 most recent forks per repo (with timestamps)

**Important**: Filter out the target user's own stars and forks. Owners often star their own repos, and including self-stars in an "external activity" dashboard is misleading. When presenting recent stars/forks, only show entries where the login differs from the target username.

For orgs, change `user(login:)` to `organization(login:)` and drop the `ownerAffiliations` filter.

## Step 3: Traffic Data (Last 14 Days)

Traffic is REST-only and requires push access. Only query the top 15 repos by most recent push. Combine clones and views into a single bash loop to keep output compact:

```bash
for repo in <repo1> <repo2> ...; do
  echo "=== $repo ==="
  gh api "repos/$repo/traffic/clones" -q '"Clones: \(.count) total, \(.uniques) unique"' 2>/dev/null
  gh api "repos/$repo/traffic/views" -q '"Views: \(.count) total, \(.uniques) unique"' 2>/dev/null
done
```

If a request returns 403 (no push access), skip silently -- the `2>/dev/null` handles this.

For repos with 10+ views, also fetch top referrers (how people are finding the repo):
```
gh api "repos/<full_name>/traffic/popular/referrers" \
  -q '.[] | "\(.referrer): \(.count) views (\(.uniques) unique)"'
```

Sort traffic results by total clones + views descending.

## Step 4: External Issues and PRs

Only query repos that have open issues (known from Step 2's `issues.totalCount`). This avoids wasting API calls on repos with zero external activity.

```bash
for repo in <repos-with-open-issues>; do
  echo "=== $repo ==="
  gh api "repos/$repo/issues?state=all&sort=updated&per_page=15&direction=desc" \
    -q '.[] | select(.user.login != "<username>" and .user.login != "dependabot[bot]") | [.updated_at[0:10], (if .pull_request then "PR" else "Issue" end), "#\(.number)", .title, .user.login, .state] | @tsv' 2>/dev/null
done
```

Only include issues updated in the last 12 months. Group by repo.

Present issues and PRs in separate groups:
- **Open issues** from humans (not bots) -- these need attention
- **Open PRs** from humans -- these may need review
- **Dependabot PRs** -- summarize as a count per repo (e.g., "hookflow: 6 open dependabot PRs") rather than listing each one, since they're noisy

Also check for dependabot PRs separately if any repos have them:
```bash
gh api "repos/$repo/issues?state=open&per_page=100&direction=desc" \
  -q '[.[] | select(.pull_request and .user.login == "dependabot[bot]")] | length' 2>/dev/null
```

## Step 5: Recent Releases

Only check repos pushed to in the last 3 months (known from Step 2's `pushedAt`):

```bash
for repo in <recently-pushed-repos>; do
  result=$(gh api "repos/$repo/releases?per_page=3" \
    -q '.[] | [.published_at[0:10], .tag_name, .name // .tag_name] | @tsv' 2>/dev/null)
  [ -n "$result" ] && echo "=== $repo ===" && echo "$result"
done
```

## Step 6: Present Dashboard

Format as a compact dashboard. Omit any section that has no data.

```
=== GitHub Activity: <username> ===
Public repos: N (non-fork, non-archived)

--- Top Repos by Stars ---
  Repo                Stars  Forks  Open Issues
  oxbow                 169     65            1
  markdown-pad-fx        35      4            0
  ...

--- Traffic (Last 14 Days) ---
  Repo              Clones (uniq)  Views (uniq)
  cc-marketplace       23 (17)       15 (5)
  jarvis-mcp           19 (13)       22 (19)
  ...

  Top referrers for <repo>:
    google.com: 15 (8 unique)
    github.com: 10 (6 unique)

--- Recent Stars (last 6 months) ---
  oxbow: @user1 (2026-03-01), @user2 (2026-02-15)
  gandalf: @user3 (2026-03-10)

--- Recent Forks (last 6 months) ---
  jarvis-mcp: @savelii (2026-01-29)

--- Open Issues from Others ---
  oxbow:
    #78 Sort icon not coming - @Shubham2202 (2023-09-05)
  highlight.fx:
    #9 Add library entry to JFXCentral - @dlemmermann (2021-11-10)
    #5 Add CSS Syntax - @dlemmermann (2021-08-25)

--- Open PRs from Others ---
  rustic-git:
    #14 Get ahead/behind count - @lukeflo (2025-11-05)

--- Dependabot ---
  hookflow: 6 open PRs

--- Recent Releases ---
  gandalf: v2.1.0 (2026-03-01)
```

Use inline bar charts (unicode block chars) for traffic visualization when there are enough data points to make them useful.

End with a **Highlights** section calling out:
- **Gaining traction**: repos with new stars or forks in the last month
- **Needs attention**: unanswered external issues or PRs, especially open ones older than 30 days
- **Discovery opportunity**: repos with high traffic but low stars (suggests people find it useful but don't star -- a better README or call-to-action might help)
