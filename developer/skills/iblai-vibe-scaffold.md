# /iblai-vibe-scaffold

> Mirrored from [`iblai/vibe`](https://github.com/iblai/vibe) · [`skills/start/iblai-vibe-scaffold/SKILL.md`](https://github.com/iblai/vibe/blob/main/skills/start/iblai-vibe-scaffold/SKILL.md). This page is generated — edit it in the repository, not here.

> **First time here?** If `iblai.env` has no `ARCHITECTURE=`, run `/iblai-vibe-start` first (four questions; two minutes) — it decides single-org / multi-org / headless and who signs in, and every skill reads the answer.

How an ibl.ai app gets its skeleton: the **project templates** that scaffold
a new app or add a feature, and the steps that assemble them. The templates
here record what a generated app contains; the references document the
assembly steps so the skills can perform them directly — render the
`assets/` templates, then apply the patches described in each reference.

> Do NOT add custom styles / colors to ibl.ai SDK components — they ship
> with their own styling. See
> [BRAND.md](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/BRAND.md).

> **Common setup (env, conventions, verification):** see
> [docs/skill-setup.md](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/docs/skill-setup.md).

## Two ways in

| Path | How | Result |
|---|---|---|
| **New app** | scaffold from **vibe-starter** (bundled with [`/iblai-vibe-ops-init`](/developer/skills/iblai-vibe-ops-init)), or render the `base`+`agent` templates | A complete app — auth, providers, Redux store, a chat page, Tauri-ready |
| **Existing app** | [`/iblai-vibe-auth`](/developer/skills/iblai-vibe-auth), then the feature skills | ibl.ai features layered onto a vanilla Next.js project |

Both map `PLATFORM`/`TOKEN`/`DOMAIN` from `iblai.env` into the
`NEXT_PUBLIC_*` vars in `.env.local` (see
[`references/config-command.md`](https://github.com/iblai/vibe/blob/main/skills/start/iblai-vibe-scaffold/references/config-command.md)).

> **Prefer the starter for greenfield work.** For brand-new projects,
> `vibe-starter` (the pre-wired template bundled with `/iblai-vibe-ops-init` —
> see the repo CLAUDE.md "Getting Started") is the recommended path; the
> `base`+`agent` templates here are the manual equivalent.

## Env: `.env.local` before build

A vibe app (scaffolded from **vibe-starter** or rendered from these templates)
reads its config from `.env.local`. **Always ensure one exists before
`pnpm build`** — a missing file is the usual cause of a mis-tenanted build.
(vibe-starter apps carry the URL defaults in code, so only the organization key
and `IBLAI_API_KEY` are at stake; template-rendered apps need the full set):

```bash
[ -f .env.local ] || cp .env.example .env.local
```

No `.env.example` in the project? Seed one from this skill's canonical copy
(the vibe-starter `.env.example`): [`assets/env.example`](https://github.com/iblai/vibe/blob/main/skills/start/iblai-vibe-scaffold/assets/env.example).
Then map `PLATFORM` → `NEXT_PUBLIC_MAIN_TENANT_KEY` and `TOKEN` →
`IBLAI_API_KEY` from `iblai.env` (see
[`references/config-command.md`](https://github.com/iblai/vibe/blob/main/skills/start/iblai-vibe-scaffold/references/config-command.md)).

## The templates

The Jinja2 (`.j2`) templates live as assets beside this skill. They use
`{{ variable }}` placeholders you fill in when rendering
(platform key, app name, agent id, …) — see
[`references/template-system.md`](https://github.com/iblai/vibe/blob/main/skills/start/iblai-vibe-scaffold/references/template-system.md) for the
variable contract.

- **[`assets/base/`](https://github.com/iblai/vibe/tree/main/skills/start/iblai-vibe-scaffold/assets/base/)** — the non-shared foundation:
  `app/(app)/{layout,page}.tsx`, `providers/index.tsx`, `store/index.ts`,
  `lib/iblai/{auth-utils,config}.ts`, `lib/config.ts`, `next.config.ts`,
  `components/ui/{button,sonner}.tsx`, `package.json`, `.env.example`.
- **[`assets/shared/`](https://github.com/iblai/vibe/tree/main/skills/start/iblai-vibe-scaffold/assets/shared/)** — rendered into **every** app
  (base *and* agent): root `app/layout.tsx`, the `(auth)/sso-login-complete`
  callback, `components/app-shell.tsx`, `providers/store-provider.tsx`,
  `hooks/use-user.ts`, `lib/{utils,hooks}.ts`, `CLAUDE.md`, `.mcp.json`,
  `public/env.js`, the tooling configs (eslint/tsconfig/tailwind/postcss/
  vitest), and the Playwright `e2e/` suite.
- **[`assets/agent/`](https://github.com/iblai/vibe/tree/main/skills/start/iblai-vibe-scaffold/assets/agent/)** — the agent-app overlay on top of
  base+shared: a chat `app/(app)/page.tsx`, `lib/config.ts`,
  `components.json`, `package.json`, `.env.example`.

The Tauri desktop/mobile shell templates (`src-tauri/`, CI workflows)
that the build skill renders live with it:
[`iblai-vibe-ops-build/assets/tauri/`](https://github.com/iblai/vibe/tree/main/skills/ship/iblai-vibe-ops-build/assets/tauri/).

Per-feature templates (auth, account, analytics, chat, notification,
profile) live with **their** skill's assets, not here — e.g.
[`iblai-vibe-auth/assets/`](https://github.com/iblai/vibe/tree/main/skills/start/iblai-vibe-auth/assets/),
[`iblai-vibe-account/assets/`](https://github.com/iblai/vibe/tree/main/skills/organizations/iblai-vibe-account/assets/). Icons live in
[`iblai-vibe-ops-build/assets/icons/`](https://github.com/iblai/vibe/tree/main/skills/ship/iblai-vibe-ops-build/assets/icons/).

## Feature skills

Each feature has its own skill that renders its assets and applies its
patches: `/iblai-vibe-auth` (SSO, store, providers — run first), then
`/iblai-vibe-agent-chat`, `/iblai-vibe-profile`, `/iblai-vibe-account`,
`/iblai-vibe-analytics`, `/iblai-vibe-notification`, `/iblai-vibe-ops-build`,
and the rest of the `/iblai-vibe-*` family.

## Provider chain (what `base` wires up)

```
AuthProvider > TenantProvider > {children}
```

`initializeDataLayer(dmUrl, lmsUrl, legacyLmsUrl, storageService, httpErrorHandler)`
is called with 5 args (data-layer v1.2+). Redux Toolkit is deduplicated via
a webpack `resolve.alias` in `next.config.ts` — without it, SDK components
bind a different `ReactReduxContext` and RTK Query hooks return `undefined`.
The `base` templates encode all of this; reproduce them faithfully if
scaffolding by hand.

## References

- [`references/add-command.md`](https://github.com/iblai/vibe/blob/main/skills/start/iblai-vibe-scaffold/references/add-command.md) — adding a feature by hand — the render→patch contract.
- [`references/config-command.md`](https://github.com/iblai/vibe/blob/main/skills/start/iblai-vibe-scaffold/references/config-command.md) — `.env.local` reference — known variables and defaults.
- [`references/template-system.md`](https://github.com/iblai/vibe/blob/main/skills/start/iblai-vibe-scaffold/references/template-system.md) — the Jinja2 template/variable contract.

## Related skills

- [`/iblai-vibe-auth`](/developer/skills/iblai-vibe-auth) — SSO auth, store, providers (run first).
- [`/iblai-vibe-ops-build`](/developer/skills/iblai-vibe-ops-build) — Tauri desktop/mobile shell.
- [`/iblai-vibe-ops-deploy`](/developer/skills/iblai-vibe-ops-deploy) — deploy to Vercel.
- [`/iblai-vibe-ops-init`](/developer/skills/iblai-vibe-ops-init) — update a project's CLAUDE.md with platform guidance.
