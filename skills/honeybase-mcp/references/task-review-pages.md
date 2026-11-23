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

## Calling your endpoints from a page

A page reads and acts only through the org's own endpoints, as the person
viewing the task. It has no network access of its own; Honeybase makes the call.

1. **Grant the endpoint on the page.** Do this in the Create/Update Task node
   (the page card's "What this page may do"), in the task's Manage pages modal,
   or with `pages[].grants` on `create_task`/`update_task`:
   ```json
   {"endpoints": [{"endpointId": "<process public id>", "pin": "{\"id\": 42}"}]}
   ```
   - `confirm` defaults to false: the page calls without asking. Set
     `"confirm": true` when the viewer should approve each call in trusted
     Honeybase chrome. Turning it on applies to tasks that are already open:
     the viewer is asked at the page's next call.
   - `pin` is a JSON object whose top-level fields overwrite the page's body.
     For example, `{"id": 42}` keeps the page to that one record. On a node it
     may use `{{ }}`.
   - Granting needs the Build workflows permission. A grant is never read from
     the page's HTML.
2. **Use `window.honeybase` from the page's script:**
   ```js
   await honeybase.ready();                    // the bridge is open
   const ctx = await honeybase.context();
   // ctx.task {id, name, completed, formSubmitted}, ctx.page {name},
   // ctx.viewer {name}, ctx.locale, ctx.theme ("light" | "dark"),
   // ctx.endpoints [{id, name, confirm}]
   const r = await honeybase.endpoints.call(ctx.endpoints[0].id, { id: 42 });
   if (r.ok) render(r.data);                   // r.status, r.body, r.contentType, r.data (parsed JSON)
   honeybase.task.onChange(t => update(t));    // now, and whenever the task changes
   honeybase.openLink("https://example.com");  // the viewer confirms in Honeybase chrome
   ```
   Refusals reject with an error whose `code` is one of `not_granted`,
   `declined`, `rate_limited`, `too_large`, `unsupported`, `refused` or
   `closed`.

Calls run as the viewer: `Trigger.result.caller.kind` is `"task_page"` and
`Trigger.result.caller.task` is `{taskId, pageId}`, where `taskId` is the task's
public id. Honeybase sets both; the page cannot change them. Calls are limited
to 64 KiB in and 512 KiB out, and to 60 per viewer per minute. Only
synchronous endpoints can be called. If a page navigates away from its first
document, it loses the bridge until the viewer reloads it.

## Approve or reject from the page

A page completes its task the same way: it calls an endpoint, and the endpoint
submits the task's form. The submission is exactly a person's: the answers land
in `task.form.data`, the submitter is the viewer, a form with `autoComplete`
completes the task, and the Task Update and Form Submission triggers fire, so
any workflow that already reacts to the decision keeps working unchanged. The
native Approve/Reject card stays visible as a fallback, and every open view of
the task updates live.

1. **The endpoint.** `create_process` with `kind: "endpoint"`, then
   `save_workflow_draft` with three nodes connected in order, and
   `release_workflow`:
   ```json
   {"Trigger": {"WebhookTriggerData": {}},
    "SubmitReview": {"UpdateTaskData": {
      "taskId": "{{ Trigger.result.caller.task.taskId }}",
      "formAnswers": "{\"decision\": {{ Trigger.result.body.decision }}, \"notes\": {{ Trigger.result.body.notes ?? '' }}}"
    }},
    "Answer": {"ResponseData": {"body": "{\"ok\": true}"}}}
   ```
   - `formAnswers` is a JSON template: each `{{ }}` slot becomes a JSON value
     (a string stays a string). One `{{ }}` that returns an object works too.
   - The answers must be questions of the task's form. For the built-in
     approval form (`formAttachment.presentation` `Approval` or
     `ApprovalWithNotes`) they are `decision` (`"approved"` or `"rejected"`)
     and `notes`.
   - The answers must pass what the form enforces in a browser: required
     questions answered and choice answers from the list, so
     `{"decision": "approve"}` fails. The node then writes nothing.
   - The run needs a person: the node fails when nothing but a schedule,
     trigger, API key or public caller started it, and inside a process
     reached through Call Process. A form is submitted once; a second call
     fails with "This form has already been submitted".
   - Optionally set a contract with `update_process_settings`
     (`endpointRequestSchema`), for example `decision` as an enum.
2. **The grant.** Grant the endpoint on the task's page as above. No `pin` is
   needed, because the endpoint takes the task from `caller.task`.
3. **The page:**
   ```html
   <div id="draft"></div>
   <textarea id="notes" placeholder="Notes"></textarea>
   <button id="approve">Approve</button> <button id="reject">Reject</button>
   <p id="state"></p>
   <script>
     const SUBMIT = "<Submit review endpoint public id>";
     const buttons = [approve, reject];
     honeybase.task.onChange(t => {
       const done = t.formSubmitted || t.completed;
       buttons.forEach(b => (b.disabled = done));
       state.textContent = done ? "Decision recorded." : "";
     });
     async function decide(decision) {
       buttons.forEach(b => (b.disabled = true));
       try {
         const r = await honeybase.endpoints.call(SUBMIT, { decision, notes: notes.value });
         if (!r.ok) throw new Error(r.body || "Failed");
       } catch (e) {
         state.textContent = "Could not record the decision: " + e.message;
         buttons.forEach(b => (b.disabled = false));
       }
     }
     approve.onclick = () => decide("approved");
     reject.onclick = () => decide("rejected");
   </script>
   ```
   The page does not need to refresh anything: `task.onChange` fires when the
   submission lands.

To show data in the same page (a draft, an order), add a second, read-only
endpoint and grant it with a `pin` naming the record.

## Runtime and security

HTML runs on a separate honeybase.io host, in an opaque sandbox with scripts
but without forms, popups, downloads, storage access or top navigation.
Inline scripts/styles are allowed. eval, external scripts, nested frames,
workers and fetch/XHR/WebSocket are blocked, unless the page is granted
**Public network access**. That grant (`grants.publicNetwork`, off by default)
allows fetch to any HTTPS address and HTTPS scripts. Images can use `data:`, `blob:` or
HTTPS. Stylesheets may load from fonts.googleapis.com; fonts may load from
fonts.gstatic.com or data URLs. Inline SVG is supported as markup; SVG uploads
remain prohibited. Relative network assets will not work; use HTTPS URLs.

Links propose a URL in trusted Honeybase chrome, where the viewer can choose
Open link; opened links have no opener or referrer. The page cannot access the
product DOM/session, and it can submit or complete the task only through an
endpoint it was granted. HTML cannot enlarge its iframe or cover the trusted
decision controls.

**This is not an exfiltration-proof sandbox.** Hostile scripts can navigate their
own iframe and encode data in HTTPS image URLs. Opening a page can expose the
viewer's IP/user agent to image and Google Fonts hosts. Never put secrets in
HTML. Only expose data intended for that page, including what granted
endpoints return to it. Public network access lets page code send anything it
holds to any HTTPS host. Neither HTML nor a prompt embedded in it can grant
capabilities.

Updates create immutable revisions until task deletion. History stores page
metadata only and can open old revisions in the same sandbox. Render links
expire in 60 seconds and are single-use; Reload mints a new link.

