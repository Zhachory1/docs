---
id: redmine-issue-tracking-research
status: in review
owner: Zhachory1
kanban_card: t_8dbf7367
pull_requests:
  - https://github.com/Zhachory1/docs/pull/5
---

# Redmine REST API & Hermes integration — public-source research notes
Scope: design research only, generic conclusions, no local/private sources used.
All claims below are sourced inline with URL and the date I read it (2026-11-02, UTC).

======================================================================
1. Redmine REST: issue create/update/search; API keys; permissions
======================================================================

Create / update
- Create: POST /issues.[format] with an `issue` hash (project_id, tracker_id,
  status_id, priority_id, subject, description, category_id, fixed_version_id,
  assigned_to_id, parent_issue_id, custom_fields, watcher_user_ids, is_private,
  estimated_hours). Update: PUT /issues/[id].[format], same hash plus `notes`
  (change comment) and `private_notes`.
  Source: https://www.redmine.org/projects/redmine/wiki/Rest_Issues (2026-11-02)

Search / filter
- GET /issues.[format] returns open issues only by default. Filters: issue_id
  (comma-separated), project_id, tracker_id, assigned_to_id (incl. "me"),
  status_id ("open"/"closed"/"*"/numeric id), cf_N for custom field N (supports
  "~foo" substring match), created_on/updated_on with ">=", "<=", "><" range
  operators (must be URL-hex-encoded), sort=field:asc|desc. Pagination via
  offset/limit (default 25, max 100) or page.
  Source: https://www.redmine.org/projects/redmine/wiki/Rest_Issues (2026-11-02)
- Full-text search across issues/wiki/etc.: GET /search.[format]?q=... with
  scope, issues, titles_only, attachments flags.
  Source: https://www.redmine.org/projects/redmine/wiki/Rest_Search (2026-11-02)
  Note: the API table marks "Search" as Alpha status (2026-11-02,
  https://www.redmine.org/projects/redmine/wiki/rest_api) — treat behavior as
  less stable than the Issues resource (Stable).

Authentication / per-user API keys
- REST API must be turned on: Administration → Settings → API → "Enable REST
  API". Two auth modes: HTTP Basic (login/password) or an API key. The key can
  be sent as a `key` query parameter, as the HTTP Basic username with any
  password, or as an `X-Redmine-API-Key` header (added in Redmine 1.1.0). Each
  user has their own key, visible on their /my/account page — this is what
  makes it "per-user": actions taken via the key are attributed to and scoped
  by that user's own role/permissions, not a shared service account.
  Source: https://www.redmine.org/projects/redmine/wiki/rest_api (2026-11-02)
- Newer Redmine code (seen in a core attachment/patch on redmine.org) also
  resolves a "Personal access token" in addition to the legacy API key via the
  same `key` param/header, logged as api_auth_credential "pat:<id>" vs
  "api_key" — i.e. personal access tokens are a newer, scoped alternative to
  the classic per-user API key on recent versions.
  Source: https://www.redmine.org/attachments/36640 (2026-11-02). I could not
  confirm which Redmine version introduced personal access tokens from this
  page alone — mark as uncertain, worth confirming against release notes
  before the designer assumes it's available on a given target version.

Project permissions / roles
- A Role is a named bundle of permissions (e.g. "Manage members", "Edit
  issues", "View issues"); a user can hold different role(s) per project, and
  if they hold several roles on one project the permissions union. Two
  built-in system roles, "Non member" and "Anonymous", apply only to public
  projects and cannot be bound to a specific user.
  Source: https://www.redmine.org/projects/redmine/wiki/RedmineRoles (2026-11-02)
- "Issues visibility" on a role controls whether a user sees all issues, all
  non-private issues (default), or only issues they created/are assigned to —
  relevant if a credential's user role is scoped down rather than full admin.
  Source: same as above.
- REST endpoints for roles/memberships: GET /roles.[format] lists roles; GET
  /roles/[id].[format] lists a role's permission strings (e.g. view_issues,
  add_issues, add_issue_notes); POST/PUT /projects/:project_id/memberships
  adds/updates a user's role_ids on a project.
  Source: https://www.redmine.org/projects/redmine/wiki/Rest_Roles and
  https://www.redmine.org/projects/redmine/wiki/Rest_Memberships (2026-11-02)
- Practical implication (my inference, not a direct citation): using a
  dedicated per-user API key scoped to a low-privilege role (e.g. only
  add_issues/add_issue_notes on the target project) bounds what a compromised
  key can do, consistent with how Redmine's own role/permission model is
  designed to be scoped per project.

======================================================================
2. Issue statuses, fields, custom fields, and dedupe across threads
======================================================================

Statuses / workflow
- Issue statuses are free-form, admin-defined, and can be added/removed;
  each has an "Issue closed" flag (more than one status can count as closed)
  and an optional "% Done" auto-set value. Workflow is configured per
  (role, tracker) pair as a matrix of allowed "current status → new status"
  transitions; a user needs the "Edit issues" permission to change status at
  all. Field-level overrides (read-only / required) can also be set per
  (role, tracker, status).
  Source: https://www.redmine.org/projects/redmine/wiki/redmineissuetrackingsetup
  (2026-11-02)
- REST exposes the status list read-only: GET /issue_statuses.[format]
  ("Issue Statuses" resource, marked Alpha, available since 1.3).
  Source: https://www.redmine.org/projects/redmine/wiki/rest_api (2026-11-02)

Custom fields
- Custom fields are typed (boolean, date, list, string, etc.), can be flagged
  required/for-all-projects/filterable/searchable, and are managed admin-side;
  GET /custom_fields.[format] (admin-only) returns each field's id, format,
  is_required, multiple flag, and possible_values.
  Source: https://www.redmine.org/projects/redmine/wiki/redminecustomfields and
  https://www.redmine.org/projects/redmine/wiki/Rest_CustomFields (2026-11-02)
- In issue create/update payloads, custom fields are set via a `custom_fields`
  array of {id, value}; fields with `multiple=true` take an array value, e.g.
  {"value":["1.0.1","1.0.2"],"multiple":true,"name":"Affected version","id":1}.
  Source: https://www.redmine.org/projects/redmine/wiki/rest_api (2026-11-02)
- Filtering issues by custom field value uses cf_<id> as the query parameter
  (e.g. ?cf_4=~foo for substring match).
  Source: https://www.redmine.org/projects/redmine/wiki/Rest_Issues (2026-11-02)

Dedupe across follow-up threads
- Redmine's own mechanism for "this is the same problem reported again" is an
  issue relation, not a special dedupe field. Relation types include
  "duplicates" / "duplicated" (reciprocal): if issue B duplicates A, closing A
  auto-closes B, but closing B leaves A open.
  Source: https://www.redmine.org/projects/redmine/wiki/redmineissues (2026-11-02)
- REST: POST /issues/:issue_id/relations.[format] with a `relation` hash
  {issue_to_id, relation_type (relates|duplicates|duplicated|blocks|blocked|
  precedes|follows|copied_to|copied_from), delay}. Returns 201 on success,
  422 with validation errors otherwise. A GET on issues can include relations
  via ?include=relations.
  Source: https://www.redmine.org/projects/redmine/wiki/Rest_issuerelations and
  https://www.redmine.org/projects/redmine/wiki/Rest_Issues (2026-11-02)
- As of the pages read, there is no batch/array relation-creation endpoint —
  an open feature request (#29272) asks for exactly that and is still just a
  proposal, not shipped.
  Source: https://www.redmine.org/issues/29272 (2026-11-02)
- Uncertainty / generic note: Redmine itself has no built-in "same thread"
  detection — any dedupe logic (matching a new report to an existing issue by
  subject similarity, custom-field correlation id, external thread id, etc.)
  is something a calling integration has to implement itself, then express
  the result as a "duplicates" relation and/or a note on the existing issue.
  I found no public Redmine documentation describing an automated dedupe
  feature; this is a design gap for the integration to fill, not a Redmine
  capability to rely on.

======================================================================
3. Hermes custom MCP tool patterns vs direct HTTP vs a sidecar, and
   keeping a credential out of the model's context
======================================================================

Three ways Hermes can reach an external API, per its own docs:
a) MCP server (stdio or HTTP) — declared under `mcp_servers` in
   ~/.hermes/config.yaml; Hermes connects at startup, lists the server's
   tools, and the model calls them like built-ins. A stdio server's `env:`
   map is what carries credentials into that separate process.
   Source: https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp
   and https://github.com/NousResearch/hermes-agent/blob/main/website/docs/reference/mcp-config-reference.md
   (2026-11-02)
b) A Hermes plugin — a directory under ~/.hermes/plugins/ with a manifest,
   schemas.py (what the model reads) and tools.py (handler code that runs
   in-process when the model calls the tool); `ctx.register_tool()` wires
   them together.
   Source: https://hermes-agent.nousresearch.com/docs/developer-guide/plugins
   and https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins
   (2026-11-02)
c) Direct HTTP from inside a sandboxed terminal/execute_code call the model
   issues itself (e.g. curl), vs. a separate long-running "sidecar" process
   the model talks to indirectly. Hermes' own docs frame MCP/plugins as the
   recommended alternative specifically so the model does not have to hold
   or compose the HTTP credential itself: "For most custom tool creation,
   start with plugins... The Adding Tools page is for built-in Hermes core
   development, not the usual user/custom-tool path."
   Source: https://hermes-agent.nousresearch.com/docs/getting-started/learning-path
   (2026-11-02)

Keeping the credential out of the model's context — what's actually
documented, and where the real boundary sits:
- Hermes' own published trust model is explicit that the LLM is the untrusted
  party. Its SECURITY.md states: "The only security boundary against an
  adversarial LLM is the [process/sandbox boundary]" and describes
  "Credential Scoping" (§2.3): shell subprocesses, MCP subprocesses, cron job
  scripts and the code-execution child all get a filtered environment —
  provider API keys and gateway tokens are stripped by default, and only
  variables an operator or a loaded skill explicitly declares are passed
  through.
  Source: https://github.com/NousResearch/hermes-agent/security/policy
  (also mirrored at https://github.com/NousResearch/hermes-agent/blob/main/SECURITY.md)
  (2026-11-02)
- Critically, that same document says the filtering does NOT protect against
  a compromised in-process component: "Any component running inside the
  agent process (skills, plugins, hook handlers) can read whatever the agent
  itself can read, including in-memory credentials. The mitigation ... is
  operator review before install ... not environment scrubbing." In other
  words, a Hermes *plugin* handler runs in the same process as the model's
  tool-call loop and can see any env var the agent process can see — a
  plugin is not, by itself, a credential-isolation boundary against
  Hermes-internal secrets; it is only isolated from LLM-directed shell code.
  Source: same SECURITY.md (2026-11-02)
- An MCP *stdio server*, by contrast, is a genuinely separate OS process:
  Hermes passes it only the `env:` entries declared in its config_servers
  entry, and the model only ever sees the tool's JSON return value — it never
  sees the process environment or the credential used to authenticate to the
  external API. This is the stronger isolation of the three options for a
  credential the model must never see, at the cost of running and operating
  a second process.
  Source: https://hermes-agent.nousresearch.com/docs/reference/environment-variables
  and https://github.com/NousResearch/hermes-agent/blob/main/website/docs/reference/mcp-config-reference.md
  (2026-11-02)
- Skills' `required_environment_variables` are explicitly documented as "never
  shown to the model" and are passed through only into sandboxed execution
  (terminal/execute_code), which is a different mechanism from either plugins
  or MCP — it is for letting the model's own shell/code invoke a CLI that
  needs the var, not for keeping a tool's internal HTTP call opaque to the
  model.
  Source: https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills
  (2026-11-02)
- Net comparison (generic, not a recommendation): a bare "direct HTTP via the
  model's own terminal/code tool" path requires the credential to be
  passed-through into that sandbox and is explicitly the weakest isolation
  per Hermes' own threat model, since any LLM-generated code in that sandbox
  can read a passed-through var. A Hermes plugin tool keeps the credential out
  of the model's visible context (the model only sees the tool's declared
  schema and JSON return) but does not isolate the credential from other
  in-process code per the SECURITY.md caveat above. An MCP stdio/HTTP server
  is the only one of the three that runs the credential in a genuinely
  separate process boundary from the agent's own Python runtime.

Prior art (not a recommendation, just noting existing public work — treat as
unaudited third-party code, not vetted by this research): several public
MCP servers already wrap the Redmine REST API as stdio or HTTP MCP servers,
taking a Redmine API key via environment variable (e.g.
github.com/Nikhilprasad-r/redmine-mcp-server, github.com/snowild/redmine-mcp,
github.com/mihajlovicjj/redmine-mcp-server, github.com/zacharyelston/
redmine-mcp-server). None of these were read for security behavior, only
listed as existing public implementations of the same integration shape.
Source: web search results, URLs as listed, 2026-11-02.

======================================================================
4. PostgreSQL and uploads (attachments) backups
======================================================================

PostgreSQL
- pg_dump exports a database as a script (plain SQL, restore via psql) or as
  one of the archive formats (-Fc custom, -Fd directory, -Ft tar); only the
  archive formats are restored with pg_restore, which supports selective /
  reordered / parallel restore (-j). The directory format is the only one
  that also supports parallel *dump* (-j on pg_dump itself).
  Source: https://www.postgresql.org/docs/current/app-pgdump.html and
  https://www.postgresql.org/docs/current/app-pgrestore.html (2026-11-02)
- Plain SQL-script restore: `psql -X dbname < dumpfile` (database must be
  created first, e.g. from template0). Custom-format restore: `pg_restore -d
  dbname filename`. Whole-cluster: `pg_dumpall`.
  Source: https://www.postgresql.org/docs/14/backup-dump.html (2026-11-02)

Redmine's own files (uploads/attachments) — not covered by pg_dump
- Redmine's own backup guidance is explicit that a backup needs two parts:
  the database, and the `files` directory where uploads are stored
  (`attachments_storage_path`, default `files/` under the install root; can be
  relocated via config/configuration.yml). A database-only backup leaves
  issues that reference attachments which no longer exist on restore.
  Source: https://www.redmine.org/projects/redmine/wiki/RedmineBackupRestore
  and https://www.redmine.org/projects/redmine/wiki/redmineinstall (2026-11-02)
- Recommended approach is a plain file copy/sync of that directory (e.g.
  `rsync -a .../files .../backup/files`), run alongside (not as a substitute
  for) the database dump, since there's no REST/API path to bulk-export
  attachment bytes — they're ordinary files on disk keyed by a generated
  numeric/timestamp prefix.
  Source: https://www.redmine.org/projects/redmine/wiki/RedmineBackupRestore
  and https://www.redmine.org/projects/redmine/wiki/RedmineFiles (2026-11-02)
- A third-party walkthrough (lower-trust than redmine.org's own wiki, treated
  here only as a corroborating example, not as an authoritative source)
  frames a complete backup as three parts — database, files/ directory, and
  config/configuration.yml (since it records any non-default
  attachments_storage_path) — and shows a cron'd pg_dump/mysqldump + tar +
  off-box sync pattern.
  Source: https://redminepro.com/blog/redmine-backup-restore (2026-11-02,
  vendor blog, not Redmine's own documentation — corroborating only)

======================================================================
What I could not establish
======================================================================
- Exact Redmine version that introduced "Personal access tokens" as distinct
  from the classic per-user API key (section 1) — the only public evidence
  found was a code/patch attachment on redmine.org, not a dated changelog
  entry.
- Any built-in Redmine feature for automatic duplicate/thread detection
  (section 2) — none found; treated as a design gap, not confirmed absent
  by an exhaustive search.
- Whether/how a specific target Redmine version's workflow engine blocks a
  status transition the API attempts outside the configured workflow matrix
  — the workflow docs describe the permission model but I did not find a
  worked REST example of a rejected transition (e.g. the exact error
  shape/status code).

======================================================================
Noted but not asked for
======================================================================
- Hermes' SECURITY.md explicitly disclaims any stronger boundary for plugin
  code than "operator review before install" — worth the designer's
  attention since it changes which of the three integration shapes in
  section 3 actually keeps a Redmine credential away from compromised
  in-process code, versus merely away from the model's own context window.
- The REST API table on redmine.org still marks "Issue Relations" and
  "Search" as Alpha/Beta-era statuses even on current docs pages — treat
  those two surfaces as less stable across Redmine versions than the
  "Stable" Issues resource when the designer picks what to depend on.
