# LMS (`iblai/lms`)

> **GitHub: [github.com/iblai/lms](https://github.com/iblai/lms)** — Open-source skills intelligence platform — discover courses, track competencies, earn credentials, and accelerate workforce development. Next.js 15 · React 19 · TypeScript 5.9 · Tailwind CSS 4 · Tauri 2.

## Overview

LMS is a production-ready learning and skills management platform that connects learners with courses, tracks competency growth, issues credentials, and delivers actionable analytics. It ships as a modern web application — deployed at [lms.ibl.ai](https://lms.ibl.ai) — backed by the ibl.ai multi-organization API with integrated LMS and AI agent capabilities.

In the five-repo family, LMS is a **complete sample application built on the ibl.ai backend** — like `iblai/os`, it is a full codebase you can take and modify. `iblai/vibe` is the toolkit for building apps like this one, `iblai/api` operates the platform headlessly via its REST API, and `iblai/iblai-infra-cli` deploys the backend on your own infrastructure.

Whether you're an enterprise building an internal upskilling program, a university offering online courses, or an EdTech startup creating a branded learning marketplace — LMS gives you the complete stack out of the box, on the [@iblai/iblai-js](https://www.npmjs.com/package/@iblai/iblai-js) SDK.

## Features

#### Course discovery & enrollment

Faceted search filters courses by subject, difficulty, skills, credential type, and content type, with AI-powered personalized recommendations based on learner profile and goals. Course detail pages carry rich metadata — syllabus, learning outcomes, instructor bios, prerequisites — and enrollment is flexible: self-enrollment, invitation-only, or Stripe-powered paid courses. Course content, progress tracking, and grading are delivered via the embedded edX LMS integration.

#### Skills & competency tracking

A skill inventory tracks earned skills with proficiency levels (0–5 scale) and skill points accumulated from course and unit completions. Skill leaderboards compare mastery across learners and cohorts, an onboarding flow lets learners self-report existing competencies, and skills-to-course mapping shows which courses develop which skills.

#### Credentials & badges

Learners earn and display certificates, badges, and micro-credentials, with issuer metadata, expiration tracking, and sharing for verification. Course-linked credentials are issued automatically on completion.

#### Programs & learning pathways

Multi-course programs provide structured enrollments with progress tracking, and curated pathways guide learners toward specific career goals. Program management covers pricing, enrollment windows, and custom metadata; learners and organizations can build and share custom pathways.

#### Learner profile & analytics

An activity dashboard shows courses enrolled and completed, skills acquired, and time spent, with daily/weekly time-spent charts. Learners get a shareable public profile with education, experience, and credentials, plus a resume builder for education history, work experience, and portfolio links.

#### Admin analytics

An overview dashboard covers platform-wide usage and engagement, with per-learner user analytics, topic analysis, a searchable transcript viewer, financial reporting for revenue and billing, and downloadable custom reports.

#### Enterprise & platform

Full per-organization isolation with dedicated configuration, branding, and user management; granular RBAC with roles, policies, and group-based access; SSO with configurable identity providers; Stripe billing for paid courses and subscriptions; an in-app notification system; white-labeling with custom themes, logos, and advanced CSS per organization; an embedded conversational AI agent sidebar for learner support; and configurable onboarding with skill self-assessment.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | Next.js 15, React 19, TypeScript 5.9 |
| Styling | Tailwind CSS 4, Radix UI, shadcn/ui |
| State | Redux Toolkit, React-Redux |
| Forms | React Hook Form, TanStack Form, Zod |
| Charts | Recharts |
| Animation | Framer Motion |
| PDF | react-pdf, pdfjs-dist |
| Desktop | Tauri 2 (Windows, macOS, Linux, iOS, Android shell) |
| Testing | Vitest, Testing Library, Playwright |
| SDK | [@iblai/iblai-js](https://www.npmjs.com/package/@iblai/iblai-js) — unified data layer, components, and utilities |

#### Data flow

User interactions flow through React components and custom hooks into Redux (RTK Query) and on to the ibl.ai API, via the `@iblai/iblai-js` SDK — which bundles the data layer (`/data-layer`), auth and organization utilities (`/web-utils`), and shared UI components (`/web-containers`) under a single package.

## Configuration

All app config is `NEXT_PUBLIC_*` (exposed to the browser); defaults match `.env.example`. The key required variables are `NEXT_PUBLIC_API_BASE_URL` (default `https://api.iblai.app` — the `/dm`, `/axd`, `/lms`, and `/studio` paths are derived from it), `NEXT_PUBLIC_LMS_URL` (edX LMS host), `NEXT_PUBLIC_AUTH_URL` (authentication service), and `NEXT_PUBLIC_MFE_URL` (Open edX micro-frontends host). The full variable reference is in the repo README.

#### Feature flags

Runtime features toggle via environment variables: the onboarding flow (`NEXT_PUBLIC_ENABLE_START_ROLE=true`), the embedded AI agent sidebar (`NEXT_PUBLIC_ENABLE_MENTOR=true`), RBAC (`NEXT_PUBLIC_ENABLE_RBAC=true`), course eligibility checks (`NEXT_PUBLIC_COURSE_ELIGIBILITY_ENABLED=true`), and the recommendations tab (`NEXT_PUBLIC_HIDE_RECOMMENDED_TAB=false`). The skill leaderboard is configured via organization metadata (`isSkillsLeaderBoardEnabled`), and theming is documented in the repo's `docs/theme-customization.md`.

## Deployment

#### Docker

```bash
docker build -t lms .
docker run -p 3000:3000 --env-file .env.local lms
```

#### Standalone

```bash
pnpm build
pnpm start
```

`pnpm start` runs a standard Next.js production server — deploy to any host with Node.js 25+ installed.

#### Desktop & mobile (Tauri)

The Tauri 2 shell in `src-tauri/` packages the same Next.js bundle as a native desktop app (Windows, macOS, Linux) or mobile app (iOS, Android). There are no `pnpm tauri:*` scripts — invoke the Tauri CLI directly:

```bash
# desktop dev (hot reload)
cd src-tauri && cargo tauri dev

# desktop production build
cd src-tauri && cargo tauri build

# iOS / Android
cd src-tauri && cargo tauri ios init
cd src-tauri && cargo tauri android init
```

Requirements: Rust toolchain (`rustup`), plus platform-specific dependencies (Xcode for iOS, Android SDK + NDK for Android).

## Quick Start

Prerequisites: **Node.js 25.3.0+** (nvm recommended) and **pnpm 10+** (`npm install -g pnpm`).

```bash
git clone https://github.com/iblai/lms.git
cd lms
```

```bash
pnpm install
```

```bash
cp .env.example .env.local
```

Edit `.env.local` with your ibl.ai platform URLs and feature flags, then start the dev server:

```bash
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000). Tests:

```bash
pnpm test            # vitest unit tests
pnpm test:coverage   # unit tests with coverage report
pnpm test:e2e        # playwright e2e suite (headless)
pnpm test:e2e:ui     # playwright UI mode
pnpm test:e2e:headed # playwright in a headed browser
```

To use the app you need access to an ibl.ai backend instance, which provides the Skills & Course API, edX LMS, authentication, the data platform (analytics, billing, user management, notifications), and the conversational AI API. Visit [ibl.ai](https://ibl.ai) to set up your backend or request a hosted instance.

## How It Fits With the Other Repos

LMS is one of the two complete sample applications in the ibl.ai open-source family — a working skills-intelligence product you can take and modify, built on the same backend the other repos build against, operate, and deploy.

- **[github.com/iblai/api](https://github.com/iblai/api)** — agent skills + a chat MCP server to headlessly manage and operate the entire ibl.ai platform via its REST API (`npx skills add iblai/api`)
- **[github.com/iblai/vibe](https://github.com/iblai/vibe)** — how to "vibe code" new applications on top of the ibl.ai backend (Next.js SDK + Claude Code skills; `npx skills add iblai/vibe`)
- **[github.com/iblai/os](https://github.com/iblai/os)** — the other complete sample application: the open-source AI agent platform running at os.ibl.ai, yours to fork and modify
- **[github.com/iblai/iblai-infra-cli](https://github.com/iblai/iblai-infra-cli)** — how to deploy the platform on your own infrastructure with Terraform + Ansible, given access to the backend images

## Related Documentation

- [Vibe](/developer/applications/vibe) — the toolkit for building apps like LMS
- [App CLI](/developer/applications/app-cli) — the `iblai-app-cli` scaffolding tool
- [Agent Quickstart](/developer/agents/quickstart) — create an agent and chat with it programmatically
- [RBAC](/developer/rbac/rbac) — roles, policies, and permissions
- [Notifications](/developer/applications/notifications) — the notifications system
- [Infrastructure CLI](/developer/infrastructure/infra-cli) — provisioning the backend LMS runs on
