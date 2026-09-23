# Issue tracker: GitHub

Issues and specs for this repo live as GitHub issues in `marcusraty/Brainstorm-`.

Claude Code cloud sessions have **no `gh` CLI**. Use the GitHub MCP tools (`mcp__github__*`, loaded via ToolSearch) for all operations, with `owner: marcusraty`, `repo: Brainstorm-`. If you are in a local session that has `gh`, the equivalent `gh issue …` commands are fine too.

## Conventions

- **Create an issue**: `issue_write` with `method: create`, `title`, `body`, optional `labels`.
- **Read an issue**: `issue_read` with `method: get`, then `method: get_comments` (and `get_labels` if needed).
- **List issues**: `list_issues` with `state` / `labels` filters; pass `fields` to keep responses small.
- **Comment on an issue**: `add_issue_comment`.
- **Apply / remove labels**: `issue_write` with `method: update` and the full desired `labels` list.
- **Close**: `add_issue_comment` with the closing note, then `issue_write` `method: update`, `state: closed`, `state_reason: completed` (or `not_planned`).

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external PRs as feature requests; `/triage` reads this flag.)_

## When a skill says "publish to the issue tracker"

Create a GitHub issue.

## When a skill says "fetch the relevant ticket"

`issue_read` with `method: get` and `method: get_comments`.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single issue with **child** issues as tickets.

- **Map**: a single issue labelled `wayfinder:map`, holding the Destination / Notes / Decisions-so-far / Not-yet-specified / Out-of-scope body. `issue_write` `method: create`, `labels: ["wayfinder:map"]`.
- **Child ticket**: create with `issue_write` `method: create`, `parent_issue_number: <map>` (attaches it as a GitHub sub-issue in one call). Labels: `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). To re-parent an existing issue use `sub_issue_write` `method: add` with the child's numeric **id** (not its number). Once claimed, the ticket is assigned to the driving dev.
- **Blocking**: GitHub's **native issue dependencies**, plus a `Blocked by: #<n>, #<n>` line at the top of the child body as a readable mirror. The MCP tools can't write dependencies, but the session proxy allows the REST API via `curl`: `curl -sS -X POST -H 'Content-Type: application/json' -H 'Accept: application/vnd.github+json' https://api.github.com/repos/marcusraty/Brainstorm-/issues/<child>/dependencies/blocked_by -d '{"issue_id":<blocker-db-id>}'`, where `<blocker-db-id>` is the blocker's numeric **id** (returned by `issue_write` create, or `issue_read` get), _not_ its `#number`. A ticket is unblocked when every blocker is closed.
- **Frontier query**: `issue_read` `method: get_sub_issues` on the map; keep the open ones with no assignee and no open blocker (`issue_dependencies_summary.blocked_by`, or the `Blocked by` line); first in map order wins.
- **Claim**: `issue_write` `method: update`, `assignees: ["marcusraty"]`, the session's first write.
- **Resolve**: `add_issue_comment` with the answer, then close the issue (`state: closed`, `state_reason: completed`), then append a context pointer (gist + link) to the map's Decisions-so-far.
