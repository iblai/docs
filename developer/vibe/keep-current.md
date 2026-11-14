# Keep your skills current

> Mirrored from [`iblai/vibe`](https://github.com/iblai/vibe) · [`docs/keep-current.md`](https://github.com/iblai/vibe/blob/main/docs/keep-current.md). This page is generated — edit it in the repository, not here.

This repository changes often — the SDK bumps daily, and skills are corrected as
the platform's endpoints and components move. Installed skills are a **copy**;
they do not update themselves.

- **Refresh** with the same command that installed them (idempotent):

  ```bash
  npx skills add iblai/vibe --all
  ```

  In Claude Code, `/iblai-vibe-ops-upgrade` does this and also bumps the SDK,
  then summarizes what changed.
- **Know what you have.** The latest release is
  [github.com/iblai/vibe/releases/latest](https://github.com/iblai/vibe/releases/latest);
  every change is listed in [`CHANGELOG.md`](https://github.com/iblai/vibe/blob/main/CHANGELOG.md). Your copy's age is the
  modification time of `.claude/skills/iblai-vibe/SKILL.md`.
- **Agents check for you.** The project `CLAUDE.md` that `/iblai-vibe-ops-init`
  writes asks the assistant to suggest a refresh when the installed skills are
  older than two weeks or older than the latest release.
- **Adapters** for Cursor and Codex are regenerated from the same sources on
  every change (`adapters/`), so every editor sees the same skill.
