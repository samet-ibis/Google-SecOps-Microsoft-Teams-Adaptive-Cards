# Google-SecOps-Microsoft-Teams-Adaptive-Cards
Migrating Microsoft Teams notifications in Google SecOps SOAR playbooks from plain text to interactive Adaptive Cards via a Power Automate Flow + HTTP trigger + mailbox-polling response pattern.


# Sending Adaptive Cards from Google SecOps SOAR via Power Automate

Google SecOps SOAR has a Microsoft Teams integration, but it can't send Adaptive Cards — there's no native support for them yet. The built-in Teams actions only post plain text, so any approval prompt sent through them shows up as raw text or JSON that the recipient has to read by eye. To get around that, this guide covers a method built using an HTTP action together with Microsoft Graph mail, which sends a real, button-driven Adaptive Card instead and reads the user's response back through e-mail.

<img width="1592" height="1354" alt="adaptivecard" src="https://github.com/user-attachments/assets/160062e2-644c-4fcb-88a5-b17645bfc0a7" />

<img width="2126" height="1029" alt="image" src="https://github.com/user-attachments/assets/516fdad1-e86d-4cc3-8c04-972b97dd1baf" />

<img width="613" height="1288" alt="image" src="https://github.com/user-attachments/assets/19239e11-dd60-4e5e-8ade-628c5f87004b" />


## What you need before starting

- A **Power Automate premium license** — the flow uses a connector action that isn't available on the free tier.
- A license for the **Copilot/Power Virtual Agents side** that backs the Teams connector's "post card and wait for response" action — check your tenant's licensing for this, since it depends on what's already provisioned for Power Platform.
- A **mailbox account you can read via Microsoft Graph** (something like a shared SOC mailbox works well), with an **app registration** set up for it — this is what the playbook's mail search step uses to pick up the user's response.

## Setting up mailbox access (Azure AD app registration)

SOAR's mail search step reads the result e-mails through Microsoft Graph, which means it needs its own Azure AD app — separate from whatever auth the Teams side uses.

1. **Azure Portal → App registrations → New registration.** Any name works; this app exists only to read one mailbox.
2. **API permissions → Add a permission → Microsoft Graph → Application permissions → `Mail.Read`.** Application permissions (not delegated) are what let this run unattended, without a signed-in user. Click **Grant admin consent** afterward — the permission doesn't take effect without it.
3. **Certificates & secrets → New client secret.** Copy the secret value immediately; it's hidden after you navigate away.
4. Note down the **Application (client) ID**, **Directory (tenant) ID**, and the **client secret** — these three go into SOAR's Microsoft Graph Mail integration instance configuration, along with the mailbox address itself.
5. **Scope the app to just that one mailbox (recommended).** By default, an app with `Mail.Read` can read every mailbox in the tenant. Lock it down with an Exchange Online application access policy:

   ```powershell
   New-ApplicationAccessPolicy -AppId <client-id> `
     -PolicyScopeGroupId soc-mailbox@yourtenant.com `
     -AccessRight RestrictAccess `
     -Description "Limit to SOC mailbox only"
   ```

   Without this, a leaked client secret for this app would expose every mailbox in the org, not just the one SOAR actually needs.

## Setting up the Teams bot (Copilot Studio agent)

The "bot" behind the flow's **"Post adaptive card and wait for a response"** action isn't something you register through the Bot Framework — it's a Copilot Studio agent. It needs to exist and be published to Teams before the flow can use it.

**1. Create the agent.** In Copilot Studio, use **New agent**, then pick **Agent (Standard)** under "Other ways to build" — not the GitHub Copilot option, which is for a different kind of autonomous agent.

<img width="1288" height="818" alt="agent-setup-1-new-agent" src="https://github.com/user-attachments/assets/29eabdf4-e3d3-4e34-bd07-f4851c39756a" />


**2. Give it minimal instructions.** This agent isn't doing any reasoning — it's just the identity that posts and tracks the card — so the instructions field can be as simple as a placeholder line.

**2a. Turn off the open chat box.** By default, the agent's Teams page lets people type to it like a chatbot — not what you want for something that's only supposed to post cards. Fix this by editing the Teams app package directly instead of publishing straight from the UI:

1. From the agent's **Publish** screen, open the **Teams + Microsoft 365** channel settings and download its app package (a `.zip` containing `manifest.json` and the app icons).
2. Unzip it, open `manifest.json`, and find the `bots` array. Add (or set) `"isNotificationOnly": true` on the bot entry — this is the flag that removes the compose box and turns the page into a notification-only feed.
3. Bump the `"version"` field (e.g. `1.0.0` → `1.0.1`). Teams won't accept a re-upload of the same app ID at the same version.
4. Re-zip the folder (keep `manifest.json` at the root, alongside the icons — don't zip the containing folder itself) and upload it back as a custom app update in Teams (or through the Teams admin center if it's org-managed).

This is a one-time fix per agent; once `isNotificationOnly` is set, every future republish from Copilot Studio keeps it.

<img width="1649" height="558" alt="agent-setup-2-instructions" src="https://github.com/user-attachments/assets/15e031f7-2760-4e42-91fe-cfc9422d9b25" />



**3. Publish it with the Teams channel enabled.** Open **Publish**, select **Teams + Microsoft 365**, and check the box to make it available there. You don't need to enable the Microsoft 365 Copilot toggle unless you also want it discoverable inside Copilot itself — Teams availability is what the flow actually depends on.

<img width="1635" height="809" alt="agent-setup-3-publish" src="https://github.com/user-attachments/assets/620137e6-9bbc-42f1-9523-25f8be938853" />


Click **Save and publish**. Once it's done, use the **"See agent in Teams"** link shown next to the agent preview to open it and add it to your own Teams — this is what makes it installed and able to actually post cards, not just published.

Give it a few minutes before moving on — a newly published agent typically takes around 5 minutes before it shows up as a selectable option in Power Automate.

**4. Point the flow's Wait Card action at it.** Back in the flow, the Teams **"Post adaptive card and wait for a response"** action (named `Wait Card` here) is where the agent gets wired in: set **Post as** to "Microsoft Copilot Studio agent", and pick your published agent under **Agent**. If it's not in the list yet, that's the propagation delay above — wait a bit and refresh. If it's still not in the list, then be sure that the account you signed in to Power Automate have full access to Copilot Agent.

<img width="1384" height="1298" alt="agent-setup-4-wait-card-config" src="https://github.com/user-attachments/assets/3a86751a-b4c7-4f41-8f5f-26997d320509" />


While setting the flow up, Power Automate will also prompt you to authorize a **connection** for the Teams and Mail (Office 365) actions it uses — make sure to pick an account that actually has permission to post in Teams and read/send from the mailbox you're using, not just whichever account happens to be signed in.

## Wiring the flow into the playbook

The flow and the playbook don't know about each other until you connect them manually — there's no discovery step, just a URL you copy from one side and paste into the other.

1. **Save the flow once, then open its trigger.** Click into the `manual` (HTTP request) trigger at the top. After the first save, Power Automate generates a real HTTP POST URL for it — this only appears once the flow has been saved at least once.
2. **Copy that URL.** It'll look like the one in the appendix below (`https://[environment-id].environment.api.powerplatform.com/.../triggers/manual/paths/invoke?...&sig=...`) — the `sig=` part is a real, usable authorization token for this specific trigger, so handle it like a credential.
3. **Paste it into the SecOps side.** In the playbook's `HTTPV2 - Execute HTTP Request` action, set **URL Path** to that full URL.
4. **Keep the two in sync.** If you ever re-save the flow's trigger schema (add/remove/rename a field in the JSON schema), Power Automate can regenerate the URL and signature — re-copy it into SecOps if that happens, or the playbook will start getting 404s.

## Setting up the mail search action in SecOps

This is the other half of the connection — the step that reads what the flow wrote.

1. In SecOps, configure a **Microsoft Graph Mail** (or equivalent) integration instance using the **client ID, tenant ID, and client secret** from the Azure AD app registration set up earlier, plus the mailbox address itself.
2. In the playbook's mail search step, filter by **subject or body containing the case ID** — the result e-mails are plain-text enough that a substring match on `case_id=<id>` in the body works reliably.
3. **Sort by received time, descending, and take the newest match.** A single case can produce more than one e-mail (an initial response, then possibly an escalation if the flow's own timeout also fires around the same time) — you want whichever one actually reflects the final state, not the first one that happens to match.
4. Leave the **"no results" case as a expected outcome, not an error** — that's the normal state while still waiting for a response, and it's what should drive the playbook's retry logic rather than failing the whole run.

## How it fits together

In each playbook, an HTTP POST step sends a request to the flow shared below. Fill in your own values for the fields it expects (recipient, case fields, mailbox, etc.) before using it. Once that POST goes out, the flow takes over: it builds the Adaptive Card and sends it to the target user in Teams. When the user taps one of the response buttons, the flow catches that response and sends a result e-mail to the SOC mailbox. Back in the playbook, a "Search Email" action reads that mailbox and picks up the user's answer from there.

You could also do this with a fully custom Azure Bot (Bot Framework registration, your own messaging endpoint, the works) — that's the "proper" way to receive a button response directly. But this method is simpler to set up and doesn't require standing up or maintaining any bot infrastructure, since Power Automate's Teams connector already has one built in.

## The architecture

```
SecOps playbook
   │
   ├─ HTTP request step ──────► Power Automate flow (trigger)
   │                                   │
   │                                   ├─ builds the Adaptive Card
   │                                   ├─ posts it to Teams and waits
   │                                   │  (via the connector's built-in bot —
   │                                   │   no custom bot registration needed)
   │                                   │
   │                              user taps a button
   │                                   │
   │                                   ├─ flow resumes, sends a result e-mail
   │                                   │  to a monitored mailbox
   │                                   ▼
   └─ mail search step ◄────────── that mailbox
         │
         ├─ found a matching e-mail → read the decision, branch on it
         └─ nothing found yet → step fails, playbook retries later
```

Here's roughly what that looks like in Teams — the initial card, and the reminder that follows if nothing's been clicked yet (illustrative data, not a real case):
<img width="1592" height="1354" alt="adaptivecard" src="https://github.com/user-attachments/assets/245cdc25-6aa3-4ef6-8a7f-b37b8d414df0" />



The flow is triggered by an HTTP call, not by SOAR waiting on the flow directly — SOAR fires the request and moves on. Everything after that happens on the Power Automate side: it builds and posts the card using its Teams connector's native **"Post adaptive card and wait for a response"** action, which is backed by a bot Microsoft already runs for every M365 tenant. When the user taps a button, that response comes back into the *flow* — not into SOAR. The flow's next step is what turns that response into something SOAR can see: it sends an e-mail to a mailbox SOAR already has access to, with a small, fixed-format block in the body:

```
case_id=<...>
stage=initial|reminder|escalated|error
decision=<the value of the button that was pressed>
responder=<e-mail of whoever responded>
at=<timestamp>
note=<optional free-text note>
```

Back in the playbook, a mail search step looks for an e-mail matching the case ID. If the user has already responded, it's there and the `decision=` line gets parsed straight into the next condition. If not, the search comes up empty and that step **fails** — which is expected, and is what drives the playbook's retry/wait logic to try again a bit later, until either a response arrives or the flow's own timeout fires and sends an escalation e-mail instead.

This is the core trick of the whole design: SOAR never receives a push of any kind. It only ever reads its own mailbox. The flow is doing all the real-time waiting and interaction with Teams; SOAR just polls for the outcome.

Here's what that looks like inside the flow itself:

<img width="2214" height="1590" alt="annotated_flow" src="https://github.com/user-attachments/assets/f254da40-2ebd-49a2-8495-b9bf6df6df89" />


**Timeout and reminder logic** — the flow waits for `timeout_minutes + reminder_timeout_minutes` total. A parallel branch waits just `timeout_minutes`, and if there's still no response at that point, sends a reminder card. If the full window elapses with nothing, the flow posts an escalation card and e-mails SOAR a `NO_RESPONSE` result instead of a decision.

**Error handling** — the whole thing runs inside a Try/Catch scope. A genuine failure (not a timeout) gets caught, mailed as a system error, and the run is explicitly marked `Failed` — Power Automate will otherwise sometimes report a run as "Succeeded" even when a step inside it failed, which would hide the problem from anyone watching the flow's run history.

**One deliberate inconsistency** — one of the buttons displays "Not Acknowledged" but actually submits a slightly different string as its decision value. That's not a bug: the playbook's existing condition logic already expects that older string, so the visible label was corrected without touching the value it sends, to avoid breaking every downstream branch that depends on it.

## What a converted playbook looks like
<img width="2126" height="1029" alt="annotated_playbook" src="https://github.com/user-attachments/assets/baa05694-5a9b-4314-ac56-77427af97a88" />


1. **HTTP request step** — fires the call to the flow's trigger URL with the case title, ID, timeout settings, recipient, and the dynamic fields for the card. This step's job is done the moment it gets a 2xx back; it doesn't wait for a person to respond.
2. **Mail search step** — checks the mailbox for a response e-mail matching the case ID. Fails if nothing's there yet, which is the normal state while waiting.
3. **Condition step** — once an e-mail is found, branches on its decision value.
4. **Per-branch handling** — closing the alert, posting a rejection notice back through an embedded sub-workflow, or assigning/re-staging/commenting on the case for further handling, depending on the branch. A separate branch handles the no-response/escalated case.

## Testing end-to-end before rolling out

Don't wire this into a live playbook on the first try — test the flow and the mail search in isolation first:

1. **Trigger the flow directly**, not through SecOps yet — use Power Automate's own "Test" button with made-up case data, or a manual `curl`/Postman POST to the trigger URL. Confirm the card actually shows up in Teams looking the way you expect.
2. **Click a response button.** Confirm the card updates to the "response received" message, and that a result e-mail lands in the SOC mailbox with the right `case_id=`/`decision=` values in it.
3. **Run the SecOps mail search action manually** (outside a full playbook run) against that same mailbox, and confirm it actually finds the e-mail and parses the decision correctly. This is the step most likely to silently misbehave — a subject/body filter that's slightly off will just never match anything, with no obvious error.
4. **Test the no-response path too.** Trigger the flow, don't click anything, and confirm the reminder card arrives on schedule, followed by the escalation card and the `NO_RESPONSE` e-mail once the full timeout elapses.
5. Only once all four of those work on their own, wire the HTTP step into one actual pilot playbook and test it through a real (or test) case.

## Rolling this out without touching anything live

The one rule that mattered most: no existing playbook was edited directly. Every change happened on a copy:

1. Export the playbook and its Power Automate flow as packages.
2. In the exported files, swap the Teams step for an HTTP request step, keeping the original timeout and recipient settings untouched.
3. Import the result under a new name — never overwrite the original playbook.
4. Start with a small number of simple, single-step playbooks before touching anything more complex.

It's more manual effort per playbook, but it means a mistake in the new version can't break the production one, and rollback is instant because nothing original was ever modified.

## Takeaways

- The hard part of Adaptive Cards was never sending them — it's getting a response back to a system that can't receive inbound traffic.
- Power Automate's Teams connector already has its own bot; using it avoids registering and maintaining a custom one.
- E-mail, as unglamorous as it sounds, is a reliable way to bridge a webhook-based flow back to a system that can only poll.
- Exporting, modifying, and re-importing under a new name is the safest way to change behavior in a production SOC environment.

## Appendix: SecOps-Side HTTP Request Configuration

The `HTTPV2 - Execute HTTP Request` step that replaced the Teams action step in the playbook was configured as follows (the URL signature and password field below are sanitized placeholders, not real values):

```
Method: POST
URL Path: https://[environment-id].environment.api.powerplatform.com:443/powerautomate/automations/direct/cu/12/workflows/[workflow-id]/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=[REDACTED_SIGNATURE]

Headers:
{
  "Content-Type": "application/json; charset=utf-8",
  "Accept": "application/json",
  "User-Agent": "GoogleSecOps"
}

Body Payload:
{
  "password": "[REDACTED]",
  "title": "[Case.Name]",
  "case_id": "[Case.Id]",
  "timeout_minutes": 15,
  "reminder_timeout_minutes": 15,
  "send_to": "[Event.event_principal_user_emailAddresses_1]",
  "fields": [
    { "title": "Incident Time", "value": "[Alert.CreationTime]" },
    { "title": "Username", "value": "[Event.event_principal_user_userid]" },
    { "title": "Event Name", "value": "[Event.event_metadata_productEventType]" }
  ],
  "message_text": "[Functions_String Functions_2.ScriptResult]"
}

Expected Response Values: (empty)
Follow Redirects: true
Fail on 4xx/5xx: true
Base64 Output: false
Fields To Return: response_data, redirects, response_code, response_cookies, response_headers, apparent_encoding
Request Timeout: 120
Save To Case Wall: false
Password Protect Zip: true
```

A few things worth noting:

- The values inside `title`, `case_id`, `send_to`, and `fields` come from SOAR's own placeholder engine (`[Case.Name]`, `[Alert.CreationTime]`, etc.) — there's no static data on the flow side at all; every case's content is filled in by SOAR.
- The `message_text` field carries the output of a **Functions (scripting) action** that runs earlier in the playbook — meaning whenever the free-text part of the card needs more complex conditional logic or string concatenation, that's prepared in a separate scripting step and wired into this field.
- The `timeout_minutes` and `reminder_timeout_minutes` values were kept exactly as they were in the original Teams action — the concrete expression of the migration's "behaviorally nothing should change" principle.
- The `password` field is used in the flow's own validation; the real value is not shared here.

## Appendix: Power Automate Flow Definition

The full flow package — ready to import directly into Power Automate — is attached in this repo as [secops-case-approval-card.zip](https://github.com/user-attachments/files/33059112/secops-case-approval-card.zip)
. It's the same flow described above (Build_Card, Wait_Card, the reminder/escalation branches, the error-handling scope), sanitized the same way as everything else in this doc: the flow ID, tenant ID, connected bot identifier, service account, and mailbox are placeholder values. I also uploaded a sample Playbook to test it. AdaptiveCardPB.zip



**To import it:** in Power Automate, **My flows → Import → Import Package (Legacy)**, then upload the zip.

**What you need to change before it'll actually work**, all inside the imported flow's edit view:

- **`Check_Pass` condition** — compares the incoming `password` field against a hardcoded string, currently `REPLACE_WITH_YOUR_OWN_SHARED_SECRET`. Set this to whatever secret you'll have the SOAR HTTP action send, so the two match. This is the only thing protecting the trigger URL, so don't skip it.
- **Connection references (Teams and Office 365)** — Power Automate will prompt for these during import ("Review connections" screen). Point both at an account that's actually allowed to post in Teams and send/read mail, not just whichever account you're importing with.
- **`Wait_Card`, `Notify_Reminder`, `Notify_Escalated` → Agent field** — each of these three actions has a `Post as` / `Agent` pair (see the Wait Card screenshot earlier). Re-select your own published Copilot Studio agent in that dropdown for all three; don't hand-edit the underlying bot identifier, let the designer fill it in.
- **`Mail_Response`, `Mail_Escalated`, `Mail_System_Error` → `emailMessage/To`** — currently `soc-mailbox@example.com` in all three. Point these at whatever mailbox your SOAR mail-search step is actually polling.
- **`Deadline`'s timezone string** — the flow formats the deadline as `'Turkey Standard Time'`; change this to your own timezone if that's not accurate for your team.

Everything else — the card layouts, the button values, the timeout math, the error handling — can stay as-is; those aren't tied to any particular tenant.

---
