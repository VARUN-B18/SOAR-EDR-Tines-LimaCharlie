# Setup guide

[Back to README](../README.md) | [Prompts](tines-prompts.md) | [API reference](api-reference.md) | [Troubleshooting](troubleshooting.md)

Detailed notes for rebuilding the project. Use placeholders for anything secret and follow [security-notes.md](security-notes.md).

## Contents

1. [Prerequisites](#1-prerequisites)
2. [Phase 1 - Design](#2-phase-1---design)
3. [Phase 2 - Windows VM and LimaCharlie sensor](#3-phase-2---windows-vm-and-limacharlie-sensor)
4. [Phase 3 - Telemetry and detection rule](#4-phase-3---telemetry-and-detection-rule)
5. [Phase 4 - Slack, Tines, and the webhook link](#5-phase-4---slack-tines-and-the-webhook-link)
6. [Phase 5 - Alerts](#6-phase-5---alerts)
7. [Phase 6 - Review page and branching](#7-phase-6---review-page-and-branching)
8. [Phase 7 - Isolation and authentication](#8-phase-7---isolation-and-authentication)
9. [Phase 8 - Final validation](#9-phase-8---final-validation)
10. [Cleanup](#10-cleanup)

## 1. Prerequisites

- A Mac or PC that can run VirtualBox, with a Windows Server 2022 ISO
- Free accounts: LimaCharlie, Tines, Slack, and a Gmail account with 2-Step Verification
- LaZagne downloaded onto the VM (lab only)
- A terminal with curl for API testing

## 2. Phase 1 - Design

Sketch the flow first: detection, webhook, alerts, a human decision, and an isolation branch. My flowchart used colors per tool (LimaCharlie blue, Tines purple, Slack and email green, isolation actions orange, failure red) and included a status-polling loop and a failure path.

![Design flowchart](../screenshots/01-workflow-diagram.png)

## 3. Phase 2 - Windows VM and LimaCharlie sensor

### VM

1. Create a Windows Server 2022 VM in VirtualBox: 4 GB RAM, 50 GB disk, NAT network adapter.
2. Install Guest Additions.
3. Rename the computer to `addc01.varun.local` and reboot.
4. Turn off Windows Defender real-time protection (Windows Security > Virus and threat protection > Manage settings) so LaZagne can run. Do this only on this throwaway VM.

No firewall rules are needed. With NAT the VM only makes outbound connections to LimaCharlie. A cloud VM (as in the original series) needs firewall rules; a local NAT VM does not.

### LimaCharlie

1. Create a LimaCharlie organization (I named it `addc01.varun.local`).
2. Create an installation key (Sensors > Installation Keys).

![Create installation key](../screenshots/02-installation-key.png)

3. Download the Windows 64-bit EDR sensor installer.
4. On the VM, open PowerShell as administrator and run the installer with your installation key. The installer file name includes the version:

```powershell
.\hcp_win_x64_release_<version>.exe -i <INSTALLATION_KEY>
```

5. In LimaCharlie, open Sensors List and confirm `addc01.varun.local` appears and is Receiving.

![Sensor enrolled](../screenshots/03-sensor-enrolled.png)

## 4. Phase 3 - Telemetry and detection rule

### Generate telemetry

1. On the VM, download LaZagne and keep the file when SmartScreen warns about it.
2. In PowerShell, from the Downloads folder:

```powershell
.\LaZagne.exe all
```

![LaZagne execution](../screenshots/18-lazagne-execution.png)

3. In LimaCharlie, open the sensor Timeline and search for `lazagne`. You will see `NEW_PROCESS` events for `LaZagne.exe` and a `CODE_IDENTITY` event that shows the file hash.

![Timeline](../screenshots/04-timeline-event.png)

### Build the rule

1. Go to Automation > D&R Rules > New Rule.
2. Start from LimaCharlie's public "Windows Process Creation" example and adapt it.
3. Paste the detect logic from [detection/lazagne-detection-rule.yaml](../detection/lazagne-detection-rule.yaml).

![Detect logic](../screenshots/05-detection-rule.png)

4. Paste the response block (`report` with metadata and the detection name).

![Response action](../screenshots/06-response-action.png)

5. Name the rule `Varun-Lazagne-SOAR-EDR` and save. It appears as `general.Varun-Lazagne-SOAR-EDR`.

### Test the rule

1. In the rule editor, use Target Event, select a `NEW_PROCESS` event for LaZagne, and click Test Event.
2. The result shows Matched. The evaluation trace short-circuits once a condition in the `or` group is true, so only the first condition (file path ends with `lazagne.exe`) shows as true and the rest show as not evaluated.

![Rule test](../screenshots/07-rule-test.png)

3. Clear old detections, run `.\LaZagne.exe all` again, and check the Detections page for a live detection.

![Live detection](../screenshots/08-live-detection.png)

## 5. Phase 4 - Slack, Tines, and the webhook link

### Slack workspace

1. Create a workspace named `VARUN-SOAR-EDR`.

![Slack workspace](../screenshots/09-slack-workspace.png)

2. Create a public channel `#alerts`.

![alerts channel](../screenshots/10-slack-alerts-channel.png)

### Tines webhook

1. Sign up for Tines. The new interface has no blank-canvas button, so create the workflow from the AI box with prompt A in [tines-prompts.md](tines-prompts.md).
2. Tines creates the webhook and tests it.

![Webhook created](../screenshots/11-tines-webhook-created.png)

3. Click Push live. A draft webhook does not take outside traffic.
4. Copy the webhook URL from the action card (link icon, then copy icon). It contains an `external_id` token, so treat it as a secret.

### LimaCharlie Output

1. In LimaCharlie go to Outputs > Add Output > Detections > Tines.
2. Name it `VARUN-SOAR-EDR`, set the stream to `detect`, and paste the webhook URL into the destination field.

![LimaCharlie Output](../screenshots/12-limacharlie-output.png)

3. Run LaZagne again. You should see the detection in LimaCharlie and the same event in the Tines webhook Events tab.

![Webhook event](../screenshots/13-tines-webhook-event.png)

## 6. Phase 5 - Alerts

### Slack connector (custom app)

Tines now asks for a custom Slack app.

1. Go to api.slack.com/apps and create a new app (Blank app). I named it `VARUN-SOAR-EDR`.
2. Under OAuth and Permissions, add the Bot Token Scope `chat:write`.
3. Install the app to the workspace and copy the Bot User OAuth Token (starts with `xoxb-`). Keep it private.
4. In the Tines Slack connector, choose connection type Bot token and paste the token.
5. Invite the app into the channel, or the first message fails with `not_in_channel`:

```
/invite @VARUN-SOAR-EDR
```

6. Copy the channel ID from the channel details and use it in the Slack action.

![Slack test message](../screenshots/14-slack-test-message.png)

### Send Slack alert

Ask the AI to add the alert after the webhook (prompt B), then set the exact field paths (prompt F). Nine labeled fields: Title, Time, Computer, Username, Source IP, File Path, Command Line, Sensor ID, Detection Link. See [field-reference.md](field-reference.md).

![First real Slack alert](../screenshots/16-slack-alert-first-real.png)

### Send Email (Gmail SMTP)

Gmail OAuth was denied (limited preview), so:

1. Turn on Google 2-Step Verification.
2. Create an App Password at myaccount.google.com/apppasswords.
3. Enter the Gmail address and App Password in the Tines Gmail (SMTP) connector.
4. Add the email action with prompt C. Use `<br>` for line breaks.

![Email step](../screenshots/15-tines-email-step.png)

![First real email alert](../screenshots/17-gmail-alert-first-real.png)

## 7. Phase 6 - Review page and branching

1. Add the User Prompt page (prompt D): nine read-only fields plus a Yes/No boolean named `isolate`.

![User Prompt page](../screenshots/19-user-prompt-page.png)

2. Reorder so the alerts run after the review page, which lets them include a "Detection Review URL".

![Final Slack alert](../screenshots/20-slack-alert-final.png)

![Final email alert](../screenshots/21-email-alert-final.png)

3. Add the If Else action and the No branch (prompt E). After a submit, the page shows "Response recorded".

## 8. Phase 7 - Isolation and authentication

1. Add the Yes branch (prompt G): Isolate sensor (POST), Check isolation status (GET), Send isolation confirmation (Slack).
2. Do not use the built-in LimaCharlie connector. Test your key with curl first ([api-reference.md](api-reference.md)), then create the `LimaCharlie JWT` connector (Token exchange) and attach it to both HTTP steps (prompt H).
3. Run the read-only check (prompt I) before testing isolation.

The full story of what went wrong is in [troubleshooting.md](troubleshooting.md).

## 9. Phase 8 - Final validation

Final canvas:

![Final canvas](../screenshots/29-final-canvas.png)

Run two fresh detections: one answered Yes, one answered No. Results and screenshots are in section 8 of the [README](../README.md#8-end-to-end-test-results).

## 10. Cleanup

1. Rejoin the VM to the network from LimaCharlie (Rejoin Network button) or with `DELETE /v1/<SID>/isolation`.
2. Delete the LimaCharlie API key if you are finished with the project.
3. Revoke the Gmail App Password if you no longer need it.
4. Re-enable Windows Defender real-time protection, or delete the VM.
