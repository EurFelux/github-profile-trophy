# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GitHub Profile Trophy is a Deno-based service that dynamically generates SVG trophy images for GitHub user profiles. It fetches user stats via the GitHub GraphQL API (v4), computes rank-based trophies, and renders them as an SVG card. Deployed on Vercel using the `vercel-deno` runtime.

## Commands

```sh
deno task start      # Run local server (port 8080)
deno task test       # Run tests (sets ENV_TYPE=test)
deno task lint       # Lint
deno task format     # Format code
deno task debug      # Run with inspector
```

Local dev: create `.env` with `GITHUB_TOKEN1` and `GITHUB_TOKEN2`, then visit `http://localhost:8080/?username=USERNAME`.

## Architecture

**Request flow:** `api/index.ts` (Vercel entry) → `StaticRenderRegeneration` (file-based cache layer) → GitHub API fetch → `UserInfo` → `TrophyList` → `Card` → SVG response.

Key layers:

- **`api/index.ts`** — HTTP handler. Parses query params (`username`, `theme`, `row`, `column`, `rank`, `title`, `margin-w`, `margin-h`, `no-bg`, `no-frame`), orchestrates data fetching and rendering. Returns SVG for valid requests, HTML error pages otherwise.
- **`src/Services/GithubApiService.ts`** — Makes 4 parallel GraphQL queries (repos, activity, issues, PRs) with token rotation via `Retry`. Uses `GITHUB_TOKEN1`/`GITHUB_TOKEN2` for failover.
- **`src/Repository/GithubRepository.ts`** — Abstract interface for the API service.
- **`src/Schemas/index.ts`** — GraphQL query definitions.
- **`src/user_info.ts`** — Aggregates raw GitHub API responses into a flat `UserInfo` object (total stars, commits, followers, etc.).
- **`src/trophy.ts`** — `Trophy` base class with rank calculation logic. Each trophy type (Stars, Commits, Followers, Issues, PullRequest, Repositories, Reviews, Experience) is a subclass with its own `RankCondition` thresholds. Secret trophies (MultipleLang, AllSuperRank, Joined2020, AncientUser, OGUser, LongTimeUser, Organizations) have `hidden = true`.
- **`src/trophy_list.ts`** — Collection of trophies with filtering (by title, rank, hidden status) and sorting.
- **`src/card.ts`** — Composes individual trophy SVGs into a grid layout.
- **`src/theme.ts`** — 24 color themes (flat, onedark, dracula, etc.).
- **`src/StaticRenderRegeneration/`** — File-based caching layer (hash URL → cache file) with TTL-based revalidation.
- **`src/config/cache.ts`** — Optional Redis caching (singleton `CacheProvider`, enabled via `ENABLE_REDIS=true`).

**Rank system:** `SECRET > SSS > SS > S > AAA > AA > A > B > C > UNKNOWN(?)`. Defined in `src/utils.ts` as `RANK` enum and `RANK_ORDER`.

## Dependencies

All dependencies are URL imports managed in `deps.ts` (Deno style — no package.json):
- `soxa` — HTTP client for GitHub API
- `deno/std` — testing utilities (assert, mock)
- `redis` — optional Redis client

## Environment Variables

See `env-example`: `GITHUB_TOKEN1`, `GITHUB_TOKEN2`, `GITHUB_API`, `ENABLE_REDIS`, `REDIS_HOST`, `REDIS_PORT`, `REDIS_USERNAME`, `REDIS_PASSWORD`, `PORT`.

## Conventions

- Deno runtime with TypeScript, no Node.js/npm
- `deno fmt` formatting (2-space indent, double quotes in config but generally follows Deno defaults)
- Tests use `__tests__` directories co-located with source, plus `test/test.ts` at root
- Service layer uses Repository pattern with abstract classes
