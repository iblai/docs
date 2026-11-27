# Guide: adding data sources

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/guides/connect-data-sources.md`](https://github.com/iblai/ontology/blob/main/docs/guides/connect-data-sources.md). This page is generated — edit it in the repository, not here.

A data source is either a **database** (dissected into the ontology and served) or an **HTTP API**
(served as named tools only). Both are added from **Systems**.

![Systems](https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/02-systems.png)

## Before you start

1. **A read-only login.** The engine profiles tables and samples rows; the server runs the SQL in
   certified tools. Neither needs write access. Create a dedicated role:

   ```sql
   -- PostgreSQL
   CREATE ROLE ontology_ro LOGIN PASSWORD '…';
   GRANT CONNECT ON DATABASE sis TO ontology_ro;
   GRANT USAGE ON SCHEMA public TO ontology_ro;
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO ontology_ro;
   ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO ontology_ro;
   ```

   ```sql
   -- MySQL
   CREATE USER 'ontology_ro'@'%' IDENTIFIED BY '…';
   GRANT SELECT ON edxapp.* TO 'ontology_ro'@'%';
   ```

   Snowflake: a role with `USAGE` on the warehouse, database and schema, `SELECT` on the tables.

   **Grant only what an agent may read.** `SELECT ON ALL TABLES` also hands over password hashes,
   session tokens and API secrets where an application keeps them in its own tables (an LMS's
   logins table, for one). The certified tools return certified columns only, but a raw toolset
   runs any `SELECT` the login may. In place of the two `ALL TABLES` lines above (grants add up),
   name the tables, and for a table that holds secrets grant a view of its safe columns:

   ```sql
   -- PostgreSQL: these tables, and of logins only the columns in a view
   CREATE VIEW logins_safe AS SELECT id, user_id, unique_id, sis_user_id FROM logins;
   GRANT SELECT ON courses, enrollments, submissions, logins_safe TO ontology_ro;
   ```

   Do not grant columns of the table itself (`GRANT SELECT (id, …) ON logins`): the engine lists
   only what the login may read whole, so that table is left out of the ontology with no error.

   Prefer a **replica** or a reporting copy for transactional systems.

2. **Network path.** The ontology host must reach the database host and port
   ([network.md](/developer/guides/ontology-network)). Add the host's IP to the database's allow-list.

3. **Scope.** Decide which schemas and tables matter. Dissecting 61 LMS tables when five carry
   the meaning only slows compilation and adds noise.

4. **The engine reads connections added here.** A server installed with `deploy/compose.yaml`
   never sets `CONNECTION_STRINGS` ([install.md](/developer/guides/ontology-install)). An engine run from its own
   checkout reads its own `.env`: leave `CONNECTION_STRINGS` empty there. While it is set, the
   engine's ingest ignores every connection the console adds.

## Adding a database

![Add a database](https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/02-systems.png)

1. Open **Systems** and pick the engine in the catalog, or press **New System** in the sidebar.
   The catalog shows two capabilities per kind:
   - **DISSECT** the engine can read it and infer terms.
   - **SERVE** the server runtime can expose it as tools.
     The console serves PostgreSQL, MySQL, ClickHouse and Snowflake. A database the engine
     dissects but the console does not serve (Trino, Databricks, …) appears in the map but gets no
     tools.
2. Enter the **Connection URL** with the read-only login. The system's name is its database's name,
   read from the URL (`postgresql://ontology_ro:…@sis-db:5432/sis` is `sis`). It is the prefix you
   see everywhere, e.g. `sis.students`.
3. **Scope**: add the schemas (`public`) or tables (`public.students`) to include. Leave it empty
   for everything.
4. **Dissection Options**
   - _PII detection_ tags columns that look personal (email, names, identifiers) and masks their
     sample values before any sample reaches a model. Leave it on.
   - _Distinct-value probing_ samples categorical columns so terms get better names
     (`enrollments.status → REGISTERED, DROPPED, WAITLIST, WITHDRAWN`).

   Both are engine-wide switches: they apply to every system of the platform.

5. **Test Connection**. The engine tries the login and reports what it sees, in its own words.
6. **Connect & Dissect**. The console:
   - saves the password in `$ONTOLOGY_HOME/platforms/<platform>/env` on the platform's server
     (mode 0600), for serving,
   - sets the two engine switches,
   - creates the connection in the engine, which stores it in its own database (or Vault) and
     starts the ingest.

   The password now exists in exactly two places on the platform's server and nowhere else. A
   console hosted elsewhere (the pages-only build) passes it through to the server and keeps none.
   You land on **Map**.

![Map while dissecting](https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/03-map.png)

7. As the engine reads the database, its tables appear in the map clustered by system, and the
   stages advance. The engine reports no progress of its own, so the stages and the log follow what
   it has read so far.
8. Start a compile with **Re-run** on the Map, or **Run now** on **Compilation**. The terms the
   engine names float above the tables. Click any node to see what was inferred. Terms that span
   several systems (the same Student in SIS, LMS and CRM) are called out for certification.
9. The pipeline ends when the compile does. Its last line counts the tools your certified terms
   produce, which is none until you certify one on **Ontology**.

## Adding an HTTP API

![Add an HTTP API](https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/02b-systems-http-api.png)

HTTP APIs are not dissected. The runtime calls them live, as named tools, with an upstream credential
that never leaves the host.

**Ready-made systems.** The catalog lists these under **APIs**:

- **Google**: Gmail, Google Drive, Google Calendar, Google Classroom, Google Sheets and Google
  Workspace Admin.
- **Microsoft**: Microsoft Teams, Microsoft SharePoint, Microsoft OneDrive, Microsoft Outlook,
  Microsoft Entra ID, Microsoft Planner and Microsoft 365 Education.
- **Others**: Blackboard Learn, Canvas LMS, Change Healthcare, Clever, Clio Manage, OneRoster 1.2,
  SAM.gov, Socrata Open Data, Workday and Zendesk.

Click one and the form is filled: its address, or an example of one where each customer has their
own; how it signs in, with a line on where the key comes from; and a chosen set of its endpoints,
the reads ticked and the changes left for you to tick. A `POST` that only asks a question, such as
an eligibility check or a query over a dataset, is ticked too; it is still in `<api>-writes`,
because the toolsets go by method. Type your address and your credentials, then **Add & Expose**.
Every field stays yours to change, and the steps below apply as they do to an API you enter by
hand. Google's and Microsoft's systems need a setup on their side first
([Google and Microsoft](#google-and-microsoft)).

Watch: [Connect the data source, create an MCP, and connect to an agent in 3 minutes](https://www.youtube.com/watch?v=gvjs9ZLCvrc).

1. **Systems → HTTP API** tab. Enter the **Base URL**. The host refuses an address it may not
   call, in the rule's own words. An address on the host's own machine is taken outside production
   only, or on a server that answers on its own machine only (`ONTOLOGY_HOST` is `localhost`).
2. If the API has an OpenAPI 3, Swagger 2 or Google Discovery document, enter its URL and press
   **Fetch Spec**. The host fetches it under the same address rule as the base URL, with the auth
   header when the document is on the API's own address, and lists its operations as endpoints. A
   Discovery document's paths follow its `rootUrl` and `servicePath`, so those two make the base
   URL: `https://www.googleapis.com/drive/v3`, then `/files`. Each of its methods also offers the
   document-wide `fields`, which picks what an answer carries.

   A path segment may hold `@` (`users/@me`), and `:` in any segment but the first (Google's
   `…/{range}:append`). A document over 2 MB, or one that answers with a redirect, is refused
   whole; `HEAD` and `OPTIONS` operations and header parameters are skipped without a count.
   A query name may start with `$` (Microsoft Graph's `$filter`, `$top`, `$select`): agents see it
   without the `$`, and the gateway gives it back on the call. What a tool cannot carry is left out
   and counted in a notice:
   - a query parameter named like `include[]`;
   - an operation whose path is not plain segments, runs over 2,048 characters or names a value
     over 64 characters;
   - two names an agent would see alike among an endpoint's path values, query names and `body`,
     ignoring case and a leading `$` (`filter` and `$filter`): a query parameter is left out
     alone, a path value with its operation;
   - every operation past the 5,000th.

3. **Upstream Auth Header**: the header the API expects, e.g. `Authorization: Bearer <secret>`. A
   value alone, such as `Bearer <secret>`, is sent as `Authorization`, as written. The host moves
   the secret into `$ONTOLOGY_HOME/platforms/<platform>/env` and keeps `${NAME}` in its place. You
   may write `${NAME}` yourself for a secret already in `env`.

   **OAuth 2.0**, for an API that issues short-lived tokens: switch the field to **OAuth 2.0** and
   enter the **Token URL** (or its path, when it is on the base URL's address), the **Client ID**
   and the **Client Secret**, and a **Scope** if the API wants one. With a **Refresh Token** the
   host renews from that; without one it signs in with the client ID and secret alone. Tick
   **Credentials in the body** for a token URL that takes no Basic header. The host signs in once
   as you save, so a wrong secret is refused then, in the API's own words. The client secret and
   the refresh token go into `env`; the token itself is kept in memory, renewed before it ends,
   and set by the gateway on each call, so an agent neither sees it nor sends one of its own.
   Fetch Spec signs in the same way with client credentials. With a refresh token it asks for the
   document without a token, so the refresh token is not spent before the API is saved; a document
   that needs a token then answers 401 or 403, and the error says why.

4. Tick the **Endpoints to Wrap**, or name them by hand. Each becomes an `http` tool. `GET`
   endpoints go in the toolset `<api>-reads`; the rest go in `<api>-writes`, which is reached
   by a policy that names it, or by a platform admin's own credential.
5. **Add & Expose** saves the API in `$ONTOLOGY_HOME/platforms/<platform>/apis.yaml` and takes you
   to the server builder. Adding an API again under the same name replaces it; a server already
   serving it takes the change when that server is saved again.

Because APIs do not pass through the engine, their responses are not object types. If an endpoint
returns something the ontology already models (an applicant, a course), say so in the tool's
description so agents connect the two.

### Google and Microsoft

The Microsoft systems call Microsoft Graph v1.0 as an app, with client credentials. The Google
systems call Google's APIs as one account, with its refresh token.

**Microsoft**, set up once:

1. Register an app in Microsoft Entra (**App registrations**) and give it a client secret.
2. Add the Microsoft Graph **application** permissions of each system you use (below), then grant
   admin consent.
3. Click the system in the catalog and enter the **Token URL**,
   `https://login.microsoftonline.com/<directory ID>/oauth2/v2.0/token`, the **Client ID** and the
   **Client Secret**. The form fills the scope, `https://graph.microsoft.com/.default`, and ticks
   **Credentials in the body**.

One registration can serve all seven systems, with the same client ID and secret. Each system's
token carries every permission granted to the app, so give a system its own registration where
that matters.

| System                  | Reads                                                                                                                          | Writes                                               | Notes                                                                                                                                                                                                                                                                                                                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Microsoft Teams         | Team.ReadBasic.All, Channel.ReadBasic.All, TeamMember.Read.All, ChannelMember.Read.All, ChannelMessage.Read.All, Chat.Read.All | Channel.Create, TeamMember.ReadWriteNonOwnerRole.All | An app cannot post channel or chat messages: Microsoft allows it only for migration. It adds members, not owners or guests. Messages come at most 50 a page. No approval form and no billing: protected-API approval ended in 2023, and Teams APIs stopped being metered in August 2025.                                                                                              |
| Microsoft SharePoint    | Sites.Read.All                                                                                                                 | Sites.ReadWrite.All                                  | Search with app permissions needs `region`, and a single-geo organization has no `dataLocationCode`: tell agents its region code (NAM, EUR, …). Filters work on indexed columns only, because the console cannot send the `Prefer` header. Reading unapproved items in a list with content approval needs Sites.Manage.All. Sites.Selected does not work with site search.            |
| Microsoft OneDrive      | Files.Read.All                                                                                                                 | Files.ReadWrite.All                                  | Search covers shared content unless the body asks for private content too. A private index can take days to a week to build. Files.ReadWrite.All also lets the credential delete, though the system offers no delete.                                                                                                                                                                 |
| Microsoft Outlook       | Mail.Read, Calendars.Read, Contacts.Read                                                                                       | Mail.Send, Calendars.ReadWrite, Contacts.ReadWrite   | These reach every mailbox. To limit the app to some mailboxes, use RBAC for Applications in Exchange Online, which replaces application access policies. `create_event` always sends invitations. Microsoft allows 10,000 requests per 10 minutes and 4 at once, per app per mailbox. Learn pages print “Calendars.ReadBasic”; the application permission is Calendars.ReadBasic.All. |
| Microsoft Entra ID      | Directory.Read.All, AuditLog.Read.All                                                                                          | none                                                 | Sign-in logs through Graph need Microsoft Entra ID P1 or P2. Logs are kept 7 days on the free edition and 30 on P1 or P2. A user's manager is not offered: Learn marks it “Not supported” for application permissions.                                                                                                                                                                |
| Microsoft Planner       | Tasks.Read.All, or Tasks.ReadWrite.All                                                                                         | Tasks.ReadWrite.All                                  | Plans belong to Microsoft 365 groups; the Entra ID system's `list_groups` gives their IDs. Updates are not offered: Planner requires an `If-Match` header the console cannot send.                                                                                                                                                                                                    |
| Microsoft 365 Education | EduRoster.Read.All, EduAssignments.Read.All                                                                                    | EduAssignments.ReadWriteBasic.All                    | The roster comes from School Data Sync. Schools appear only when it provisions administrative units. Publishing runs in the background.                                                                                                                                                                                                                                               |

A list returns its first page only: the systems offer `filter`, `top` and `select` where Microsoft
documents them, and no next-page token.

**Google**, set up once:

1. Create or choose a Google Cloud project and enable the API of each system you use: Gmail API,
   Google Drive API, Google Calendar API, Google Classroom API, Google Sheets API, Admin SDK API.
2. On the OAuth consent screen, choose **Internal** (Workspace organizations). An External app
   needs Google's verification for restricted scopes, and while it is left in Testing its refresh
   tokens end after 7 days. **Personal** mode also skips verification and the 7-day limit, but
   serves only the account that created the project.
3. Create a **Web application** OAuth client with the authorized redirect URI
   `https://developers.google.com/oauthplayground`.
4. In the [OAuth 2.0 Playground](https://developers.google.com/oauthplayground):
   1. Open the gear, tick **Use your own OAuth credentials** and enter the client ID and secret.
   2. Choose the server-side flow and offline access.
   3. Paste the scopes of each system you use (below), authorize, then
      **Exchange authorization code for tokens**.
   4. Copy the refresh token.
   5. Remove the Playground's redirect URI from the client.
5. Click the system in the catalog and enter the **Client ID**, the **Client Secret** and the
   **Refresh Token**. The form fills the token URL, `https://oauth2.googleapis.com/token`, and
   ticks **Credentials in the body**.

One client and one refresh token holding every system's scopes can serve all six. Google renews
without a new refresh token, so the one you entered is kept. It stops working when:

- it goes unused for 6 months;
- the account changes its password, if the token carries Gmail scopes;
- the account holds more than 100 tokens for the client: the oldest is dropped, silently;
- an admin restricts the service.

From 1 February 2027, Google requires AI agents to reach Google Workspace through its MCP servers
instead of the REST APIs these tiles call, and holds agents that call the REST APIs directly to the
Standard Tier's published per-project limits, whatever tier the project chose. The Standard Tier's
5,000 requests a day per project per product is its cap on the MCP servers, not on REST (Google's
[quota usage tiers](https://developers.google.com/workspace/guides/quota-usage-tiers), last
updated 2026-11-24).

Scopes are under `https://www.googleapis.com/auth/`: gmail.modify is
`https://www.googleapis.com/auth/gmail.modify`.

| System                 | Scopes                                                                                                                                                                                                                                                                                                                                                | Notes                                                                                                                                                                                                                                         |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Gmail                  | gmail.modify (restricted)                                                                                                                                                                                                                                                                                                                             | `userId` is `me`. To reply in a thread, send `threadId` with matching `In-Reply-To` and `References` headers and the same `Subject`. Workspace's sending limit is 2,000 messages a day per user.                                              |
| Google Drive           | drive (restricted), or drive.readonly to read only                                                                                                                                                                                                                                                                                                    | drive.file does not fit: it reaches only files the app created or was given. Export is capped at 10 MB, and a Sheet exports its first sheet only. Drive returns little unless `fields` names more.                                            |
| Google Calendar        | calendar.events, calendar.calendarlist.readonly, calendar.calendars.readonly, calendar.events.freebusy                                                                                                                                                                                                                                                | `calendarId` `primary` is the account's main calendar. Free/busy times come back in UTC unless the request sets `timeZone`.                                                                                                                   |
| Google Classroom       | classroom.courses.readonly, classroom.rosters.readonly, classroom.coursework.students, classroom.announcements, classroom.topics.readonly, classroom.courseworkmaterials.readonly, classroom.guardianlinks.students.readonly, classroom.profile.emails                                                                                                | The account is a teacher (their own courses) or a Workspace for Education admin (all courses). Only the Cloud project that created a piece of coursework may grade or return its submissions. Guardians must be enabled for the domain.       |
| Google Sheets          | spreadsheets                                                                                                                                                                                                                                                                                                                                          | 60 reads and 60 writes a minute per user. A new spreadsheet lands in the account's Drive root.                                                                                                                                                |
| Google Workspace Admin | admin.directory.user.readonly, admin.directory.group.readonly, admin.directory.group.member, admin.directory.orgunit.readonly, admin.directory.device.chromeos.readonly, admin.directory.device.mobile.readonly, admin.directory.domain.readonly, admin.directory.rolemanagement.readonly, admin.reports.audit.readonly, admin.reports.usage.readonly | The account is a Workspace admin: a super admin, or delegated with the matching privileges. Activity reports arrive within minutes to hours; usage reports take days. The Drive activity report is for Business and Enterprise editions only. |

## Adding a filesystem

A filesystem is a folder, or one file, on the server's own disk, addressed
`local:///absolute/path`. It is not dissected and not drawn on the Map: the gateway serves it
itself, as two tools.

1. **Systems**, then click **Filesystem** in the catalog. The form on the right shows `local://`
   and a file picker.
2. Pick a folder or a file.
3. **Connect & Expose**.

Each filesystem becomes one toolset, `<name>-files`, ticked by default when a server is built. The
name is lowercase letters, digits and `-`, starting with a letter, 53 characters at most. A role
that reaches every read toolset reaches it too. Its two tools:

- `<name>_list_files(path?)`: one folder's entries, with name, size, modified time, and whether
  each is a folder or a file. 1,000 at most, with a `truncated` flag.
- `<name>_read_file(path, offset?, limit?)`: up to `limit` bytes of text from `offset`, 64 KiB by
  default and 256 KiB at most, answered with `size` and `next`.

The rules:

- Nothing outside `ONTOLOGY_LOCAL_ROOT` is listed or read. Unset, it is the home folder of the
  account the server runs as: `/home/ontology` on a source install ([install.md](/developer/guides/ontology-install)),
  `/opt/iblai-ontology/files` in Docker, mounted read-only (files readable by uid 1000), and your
  own home folder under `pnpm dev` or `pnpm start`. `/` is refused as a root.
- A tool's path cannot leave its own filesystem: `..`, an absolute path and a link that leads out
  are refused.
- Symlinks are not followed and special files are skipped.
- A name that starts with a dot (`.env`, `.ssh`, `.git`) is never listed or read.
- The server's own folder is refused.
- Only a server that serves one platform (`ONTOLOGY_PLATFORM` set) takes filesystems.

Text only: a file with a NUL byte is refused as "not a text file", so PDF, Office files and images
are not read. There is no recursive listing, no search inside files and no SQL over files.

## Prebuilt toolsets

Each connected database the console serves has a raw toolset, `<system>-raw`, with
`execute_sql` and `list_tables`. Tick the ones you need on the **Prebuilt** tab and the server
builder opens with them ticked. Add one only for analysts who genuinely need ad-hoc SQL, and keep it
on the read-only login. A raw toolset counts as writing, so it is reached only by a policy that
names it.

## After connecting

- **Catalog** lists every database, schema, table and column the engine read, with tags (PII) and
  the terms mapped to each.
- **Compilation** shows whether a compile is running, when the last one succeeded or failed, and
  the switches (compilation, PII detection, probing).
- **Ontology** is where you review and certify what the engine inferred before it becomes tools
  ([permissions.md](/developer/guides/ontology-permissions) covers who may certify).

## Changing or removing a source

The engine's own UI is not served on the server's host name: it opens through an SSH tunnel to the
server ([operations.md](/developer/guides/ontology-operations#the-generated-secrets)).

- Rotating a password: change it in the engine's own UI (**Settings → Connections**) and in
  `$ONTOLOGY_HOME/platforms/<platform>/env` under `<NAME>_DB_PASSWORD`, then deploy each server
  that uses it again.
- Re-scoping tables: change the connection's scope in the engine's UI and compile again; terms
  already certified are kept.
- Removing a system: delete the connection in the engine's UI. Then edit each server that used its
  toolsets, untick them and deploy again; a save that names a toolset that no longer exists is
  refused with its name.

## Troubleshooting

| Symptom                               | Likely cause                                     | Where to look                                                                                                                  |
| ------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| Test Connection fails                 | firewall or wrong login                          | the engine's message; can the host reach `db:port` with `psql` or `mysql` and the same URL?                                    |
| Connected, but no tables ever appear  | the engine's `CONNECTION_STRINGS` is set         | an engine run from its own checkout: empty it in that engine's `.env` and restart the engine                                   |
| Tables appear, no terms               | no compile has run, or it failed                 | **Compilation** page: last failure; the engine's logs on the host                                                              |
| No relationships (PostgreSQL)         | the login does not own the tables                | the engine reads declared keys only as their owner; the server's `OPENAI_API_KEY` lets it infer joins instead                  |
| Terms are poorly named                | columns are opaque codes, probing off            | turn on distinct-value probing; add column descriptions in the engine's UI and compile again                                   |
| PII column not tagged                 | detector missed it                               | tag the column `pii` in the engine's UI; tags are honoured on the next compile                                                 |
| Tool returns permission error         | login lacks SELECT on a joined table             | grant SELECT; the certified SQL shows the tables involved                                                                      |
| `503`, “No engine is connected”       | the server has no engine for this platform       | the server's operator sets `AO_URL`, or writes `$ONTOLOGY_HOME/platforms/<platform>/engine.json` ([network.md](/developer/guides/ontology-network)) |
| `503`, “ONTOLOGY_PLATFORM is not set” | the server has an engine but names no platform   | the server's operator sets `ONTOLOGY_PLATFORM` ([install.md](/developer/guides/ontology-install))                                                   |
| `403`, “serves another platform”      | the server's `ONTOLOGY_PLATFORM` names another   | connect this platform's own server; a server with an engine of its own serves one platform                                     |
| A system shows no tools               | the console does not serve that kind of database | it stays in the map without tools                                                                                              |
| A term's `search_…` tool is missing   | its name is one of the gateway's own tools       | a term named Term would make `search_terms`, the glossary's; rename the term (Enrollment Term)                                 |
