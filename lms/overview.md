# LMS Platform Overview

![The LMS home page at lms.ibl.ai, with a greeting, Explore Catalog and My Courses buttons, and a rail of course cards](/images/docs/lms/catalog/home.webp)

## What Is the LMS?

The LMS (Agentic LMS) is ibl.ai's open-source learning platform. Learners discover courses, programs, and pathways, take courses either by reading units or by learning in conversation with an AI tutor, and build a record of their skills and credentials. Course teams run their courses from the same screens, and administrators get analytics and organization-wide settings.

The platform runs in production at [lms.ibl.ai](https://lms.ibl.ai) and the source code is available at [github.com/iblai/lms](https://github.com/iblai/lms). It ships as a web application, as desktop apps for macOS, Windows, and Linux, and as mobile apps for iOS and Android.

This documentation section covers every screen of the application: navigation and onboarding, the catalog, the course player, course administration, learner profiles, analytics, notifications, and organization settings.

## Core Capabilities

#### Course Discovery & Enrollment
A searchable, filterable catalog of courses, programs, and pathways, with free, paid, and invitation-only enrollment. See [Discover](catalog/discover.md) and [Course Page](catalog/course-about.md).

#### AI-Led Courses
Courses can be taught by an AI tutor that knows the course and the unit you are on, teaches by conversation, and can decide when you have completed a unit. A **Learn / Assess** switch moves between the tutor and the unit's own assessment. See [Agent Tab](course/agent.md).

#### Course Player
The course outline, progress and grade, unit-by-unit navigation, timed exams, dates, and discussions in one place. See [Course Player](course/layout.md).

#### Course Administration
Everything a course team needs inside the course: enrollment and course-team roles, a gradebook with overrides, cohorts, due-date extensions, attempt resets, downloadable reports, analytics, credentials, and advanced settings. See [Administration](course-administration/overview.md).

#### Skills & Credentials
Earned, self-reported, and desired skills, a skill leaderboard, and credentials awarded automatically when a course is completed or passed. See [Skills](profile/skills.md) and [Credentials](profile/credentials.md).

#### Programs & Pathways
Multi-course programs with overall progress, and pathways that learners can follow or build for themselves. See [Programs](catalog/programs.md) and [Pathways](catalog/pathways.md).

#### AI Agent Everywhere
An AI agent is one click away on every page, and opens the current course's own agent when you are in a course. See [Navigating the LMS](getting-started/navigation.md#ai-agent-tab).

#### Analytics
Organization-wide dashboards for users, courses, programs, topics, transcripts, memory, AI cost, audit, and revenue, with CSV exports. See [Analytics](analytics/overview.md).

#### Onboarding
An optional Start screen that builds a learner's role and skill profile, and a configurable onboarding flow that ends by introducing an AI assistant. See [Start Screen](getting-started/start-screen.md) and [Onboarding](getting-started/onboarding.md).

#### Multi-Organization, SSO & RBAC
Each organization has its own catalog, branding, and settings, with single sign-on and role-based access control shared with the rest of ibl.ai. See [Organization Settings](organization-settings/overview.md).

## Documentation Map

#### Getting Started
[Navigating the LMS](getting-started/navigation.md) · [Start Screen](getting-started/start-screen.md) · [Onboarding](getting-started/onboarding.md)

#### Catalog
[Home](catalog/home.md) · [Discover](catalog/discover.md) · [Course Page](catalog/course-about.md) · [Programs](catalog/programs.md) · [Pathways](catalog/pathways.md)

#### Course Player
[Course Player](course/layout.md) · [Agent](course/agent.md) · [Course](course/course.md) · [Progress](course/progress.md) · [Dates](course/dates.md) · [Discussions](course/discussions.md) · [Learning Info](course/learning-info.md) · [Instructors](course/instructors.md) · [Bookmarks](course/bookmarks.md)

#### Course Administration
[Overview](course-administration/overview.md) · [Grades](course-administration/grades.md) · [Membership](course-administration/membership.md) · [Cohorts](course-administration/cohorts.md) · [Extensions](course-administration/extensions.md) · [Attempts](course-administration/attempts.md) · [Reports](course-administration/reports.md) · [Analytics](course-administration/analytics.md) · [Settings](course-administration/settings.md) · [Authoring and Proctoring](course-administration/authoring-proctoring.md)

#### Profile
[Profile Dialog](profile/profile-dialog.md) · [Gradebook](profile/gradebook.md) · [Skills](profile/skills.md) · [Credentials](profile/credentials.md) · [Activity](profile/activity.md) · [Pathways](profile/pathways.md) · [Programs](profile/programs.md) · [Courses](profile/courses.md) · [Public Profile](profile/public-profile.md)

#### Analytics
[Overview](analytics/overview.md) · [Users](analytics/users.md) · [Courses](analytics/courses.md) · [Programs](analytics/programs.md) · [Topics](analytics/topics.md) · [Transcripts](analytics/transcripts.md) · [Memory](analytics/memory.md) · [Cost](analytics/costs.md) · [Audit](analytics/audit.md) · [Monetization](analytics/monetization.md) · [Data Reports](analytics/data-reports.md)

#### Notifications
[Inbox](notifications/inbox.md) · [Alerts](notifications/alerts.md)

#### Organization Settings
[Organization Settings](organization-settings/overview.md)

#### Common Components
Shared building blocks that appear across all ibl.ai applications: [Common Components](common-components/overview.md)

## Related Resources

#### Source Code
The full application is open source at [github.com/iblai/lms](https://github.com/iblai/lms). Developer set-up, deployment, and theming are covered in the [LMS developer guide](../developer/lms.md).

#### OS
[OS](../os/overview.md) is ibl.ai's agent platform. The LMS shares its profile dialog, notifications, analytics, and organization settings, and its course agents are built and configured there.
