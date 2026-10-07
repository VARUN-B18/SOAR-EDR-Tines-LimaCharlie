# Tines prompts

[Back to README](../README.md) | [Field reference](field-reference.md) | [Troubleshooting](troubleshooting.md)

The new Tines interface is prompt-driven: you build a workflow by typing into the AI box instead of dragging actions onto a canvas. These are the prompts I used, in the order I used them.

## Contents

- [A. Create the webhook](#a-create-the-webhook)
- [B. Slack alert](#b-slack-alert)
- [C. Email](#c-email)
- [D. User Prompt page](#d-user-prompt-page)
- [E. If Else, No branch](#e-if-else-no-branch)
- [F. Fix or set exact field paths](#f-fix-or-set-exact-field-paths)
- [G. Yes branch](#g-yes-branch)
- [H. Attach the working connector](#h-attach-the-working-connector)
- [I. Read-only isolation check](#i-read-only-isolation-check)
- [J. Replay prompts](#j-replay-prompts)
- [K. Health check](#k-health-check)
- [L. Master rebuild prompt](#l-master-rebuild-prompt)

## A. Create the webhook

```
Create a workflow with a webhook trigger named "Retrieve Detections"
```

It created the webhook and tested it automatically. Click "Push live" afterwards, because a draft webhook does not accept outside traffic. Copy the URL from the action card (link icon, then copy icon). Treat it as a secret.

## B. Slack alert

```
Add a Slack action connected after the webhook that sends a message to the #alerts channel
```

When it asks what the message should contain, use prompt F.

## C. Email

```
Add a Send Email action connected after the webhook, using the same detection fields as the Slack alert, formatted with <br> line breaks for HTML
```

## D. User Prompt page

```
Add a Page action connected after the webhook, named "User Prompt", with a Boolean field called "isolate" asking "Do you want to isolate this machine?", and display the same detection fields above the question
```

## E. If Else, No branch

```
Add an If/Else action connected after User Prompt: if isolate is false, send a Slack message to #alerts saying "The computer {hostname} was not isolated, please investigate"
```

## F. Fix or set exact field paths

```
Fix the field mappings in Send Slack alert, Send Email, and User Prompt to use these exact case-sensitive paths from the webhook body: Title=cat, Time=detect.routing.event_time, Computer=detect.routing.hostname, Username=detect.event.USER_NAME, Source IP=detect.routing.int_ip, File Path=detect.event.FILE_PATH, Command Line=detect.event.COMMAND_LINE, Sensor ID=detect.routing.sid, Detection Link=link
```

See [field-reference.md](field-reference.md) for why this was needed.

## G. Yes branch

```
Add a branch after If Else for when isolate is true: make an HTTP POST request to LimaCharlie's API to isolate the sensor, using the sensor_id from detect.routing.sid. Then check the isolation status, and send a Slack message to #alerts confirming "The computer {hostname} has been isolated. Status: {isolation status}"
```

When it asks "isolate immediately on Yes?", answer yes.

## H. Attach the working connector

```
Open the Isolate sensor step and change its connector from "LimaCharlie" to "LimaCharlie JWT". Confirm which connector is now attached after the change.
```

Repeat for "Check isolation status". This was the final fix for the 401 errors: both steps were still attached to the old connector. See [troubleshooting.md](troubleshooting.md).

## I. Read-only isolation check

```
Before changing anything, run a read-only check with the "LimaCharlie JWT" connector: send GET https://api.limacharlie.io/v1/<SID>/isolation and show me the HTTP status and the isolation status field from the response. Do not isolate anything.
```

Replace `<SID>` with your sensor ID. A healthy result is HTTP 200 with `is_isolated: false`.

## J. Replay prompts

New webhook events did not automatically run the downstream steps while the workflow was being edited as a draft. These prompts replay the most recent event.

```
Run the full workflow (Send Slack alert, Send Email, and User Prompt) using the most recent Retrieve Detections event
```

```
Give me the URL to the User Prompt page for the most recent detection
```

```
Run the Isolate sensor branch using the most recent User Prompt "Yes" response
```

## K. Health check

```
Run a full health check on this workflow: confirm the "Retrieve Detections" webhook is live and has received events recently, confirm "Send Slack alert" and "Send Email" are correctly connected and using valid field paths from the most recent event, confirm "User Prompt" is reachable and its isolate field is being captured correctly, confirm "If Else" branches to the right actions for both true and false, and confirm "Isolate sensor" and "Check isolation status" are using the "LimaCharlie JWT" connector and not returning any auth errors. Report back exactly what's working and what isn't.
```

## L. Master rebuild prompt

This single prompt rebuilds the same structure from scratch. I compared a workflow built from it against the real one and the topology matched (only step names differed). Note that the first version of this prompt put the email and Slack steps in parallel with the review page; the final workflow puts them after "User Prompt" so the alerts can include the review URL.

```
Build a complete SOC detection-and-response workflow with the following steps:

1. A webhook trigger named "Retrieve Detections".

2. Connected directly after the webhook, in parallel, create these three actions:
   a. "User Prompt" - a Page action displaying nine labeled detection fields (Title, Time, Computer, Username, Source IP, File Path, Command Line, Sensor ID, Detection Link) above a Boolean field called "isolate" asking "Do you want to isolate this machine?"
   b. "Send Email" - an email action sending the same nine labeled fields, formatted with <br> line breaks for HTML, to a recipient I will specify
   c. "Send Slack alert" - a Slack action posting the same nine labeled fields to the #alerts channel

Use these exact case-sensitive field paths from the webhook body:
- Title (top-level field: cat)
- Time (detect.routing.event_time)
- Computer name (detect.routing.hostname)
- Username (detect.event.USER_NAME)
- Source IP (detect.routing.int_ip)
- File Path (detect.event.FILE_PATH)
- Command Line (detect.event.COMMAND_LINE)
- Sensor ID (detect.routing.sid)
- Detection Link (top-level field: link)

3. Connected after "User Prompt", an "If Else" action:
   - If isolate is false, connect to a "Send investigation alert" action that sends a Slack message to #alerts: "The computer {hostname} was not isolated, please investigate"
   - If isolate is true, continue to step 4

4. "Isolate sensor" - HTTP POST to https://api.limacharlie.io/v1/{sid}/isolation using sid from detect.routing.sid.
5. "Check isolation status" - HTTP GET to the same URL.
6. "Send isolation confirmation" - Slack message to #alerts: "The computer {hostname} has been isolated. Status: {isolation status}"

For steps 4 and 5 do not use any built-in LimaCharlie connector template. Tell me to manually create an "API key" connector using Token exchange authentication (Token URL https://jwt.limacharlie.io, form-encoded body with oid and secret, token path jwt, allowed for URLs starting with api.limacharlie.io) and wait for me to provide it before attaching it.

Push the workflow live once built.
```
