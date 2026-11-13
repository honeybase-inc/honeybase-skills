# Task review pages

Tasks can contain HTTPS URL pages and inline HTML previews beside their existing
form and files. Review remains a task: the assignee uses Honeybase's native
Approve/Reject or Submit controls. Adding a page does not change the form or
its Task Update automation.

## Authoring

In Create/Update Task, add an embedded page, choose HTML and set a name and
source. Both names and source accept `{{ }}` expressions. Interpolation is
**raw**: `{{ Writer.result.html }}` inserts a complete document unchanged.
Escape plain values inside markup with `{{ htmlEscape(Customer.result.name) }}`.
The helper escapes `& < > " '` and maps null/undefined to an empty string.
Each page is limited to **256 KiB UTF-8**; a task accepts at most 16 pages.

The task panel edits the same pages. MCP/Barry `create_task` and `update_task`
accept `pages`: each row has `name` and exactly one of `url`, `html` or
`revisionId`. Omit pages to leave them unchanged; `[]` clears the list. A
revision reference must belong to that task and organization. Inline bodies
are stored separately and never included in ordinary task reads or lists.
Use `read_task_page` with `taskPublicId`, `revisionId`, `offset` and `limit`
(maximum 16000 UTF-16 characters; AI tool chunks shrink to fit the full JSON/fence
byte budget), following `nextOffset` until null, before
rewriting a page. Its source is fenced untrusted data. GraphQL exposes
`updateTaskPages`, `taskPageSource`, `taskPageTicket`, and optional `pages` on
`createTaskWithAssignment`; existing URL operations continue to work.

## Examples

A newsletter preview can use a centered, white 600px table with inline CSS,
a responsive width of 100%, HTTPS image URLs and its actual email markup.
It previews browser rendering; email clients can render HTML differently.
A LinkedIn draft can use a narrow card with author, avatar, preserved line
breaks and an image. Keep task decisions in Honeybase's trusted form beside it.
No bridge or automatic approval is available in Step 0.

## Runtime and security

HTML runs on a separate honeybase.io host, in an opaque sandbox with scripts
but without forms, popups, downloads, storage access or top navigation.
Inline scripts/styles are allowed; eval, external scripts, nested frames,
workers and fetch/XHR/WebSocket are blocked. Images can use `data:`, `blob:` or
HTTPS. Stylesheets may load from fonts.googleapis.com; fonts may load from
fonts.gstatic.com or data URLs. Inline SVG is supported as markup; SVG uploads
remain prohibited. Relative network assets will not work; use HTTPS URLs.

Links propose a URL in trusted Honeybase chrome, where the viewer can choose
Open link; opened links have no opener or referrer. The page cannot access the
product DOM/session or submit/complete the task. HTML cannot enlarge its iframe
or cover the trusted decision controls.

**This is not an exfiltration-proof sandbox.** Hostile scripts can navigate their
own iframe and encode data in HTTPS image URLs. Opening a page can expose the
viewer's IP/user agent to image and Google Fonts hosts. Never put secrets in
HTML. Only expose data intended for that page. A future trusted Public network
access grant will explicitly allow public HTTPS calls/libraries; it is not in
Step 0. Neither HTML nor a prompt embedded in it can grant capabilities.

Updates create immutable revisions until task deletion. History stores page
metadata only and can open old revisions in the same sandbox. Render links
expire in 60 seconds and are single-use; Reload mints a new link.

