# SOAR + EDR Automation: LaZagne Detection, Human Approval, and Automatic Isolation

A home-lab SOC project that connects an EDR (LimaCharlie), a SOAR platform (Tines), Slack, and Gmail into one pipeline. When a credential-dumping tool (LaZagne) runs on a Windows Server 2022 VM, the pipeline detects it, alerts the analyst in Slack and email, asks "Do you want to isolate this machine?", and if the answer is Yes, isolates the machine through the LimaCharlie API and confirms the result in Slack.

This project follows the MyDFIR "SOAR EDR project" series (see [Credits](#15-credits)) and documents a local VirtualBox Windows Server 2022 lab using the newer prompt-driven Tines interface, Slack, Gmail SMTP, and LimaCharlie API authentication.

> **Lab use only.** LaZagne was run only on my own throwaway test VM. Do not run credential-dumping tools on systems you do not own or have written permission to test.

---

## Find things fast

| I want to... | Go to |
|---|---|
| See what the pipeline does, with screenshots | [Walkthrough](#5-walkthrough) |
| See the final result (isolation working) | [End-to-end test results](#8-end-to-end-test-results) |
| Understand the flow | [Architecture](#2-architecture) |
| Read the detection rule | [Detection rule](#6-detection-rule) and [the YAML](detection/lazagne-detection-rule.yaml) |
| See the MITRE ATT&CK mapping | [MITRE ATT&CK mapping](#mitre-attck-mapping) |
| Rebuild it myself | [docs/setup-guide.md](docs/setup-guide.md) |
| Copy the Tines prompts | [docs/tines-prompts.md](docs/tines-prompts.md) |
| Test the LimaCharlie API with curl | [docs/api-reference.md](docs/api-reference.md) |
| Look up the webhook field paths | [docs/field-reference.md](docs/field-reference.md) |
| Fix an error I hit | [Troubleshooting](#9-troubleshooting) |
| See every screenshot | [docs/screenshot-index.md](docs/screenshot-index.md) |
| Know what the project cannot do | [Limitations](#11-limitations-and-next-steps) |

---

## Table of contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Tools and environment](#3-tools-and-environment)
4. [Repository layout](#4-repository-layout)
5. [Walkthrough](#5-walkthrough)
   - [Phase 1 - Design](#phase-1---design)
   - [Phase 2 - LimaCharlie and the Windows VM](#phase-2---limacharlie-and-the-windows-vm)
   - [Phase 3 - Telemetry and the detection rule](#phase-3---telemetry-and-the-detection-rule)
   - [Phase 4 - Slack workspace and Tines webhook](#phase-4---slack-workspace-and-tines-webhook)
   - [Phase 5 - Alerts: Slack and email](#phase-5---alerts-slack-and-email)
   - [Phase 6 - Human decision page and branching](#phase-6---human-decision-page-and-branching)
   - [Phase 7 - Isolation through the LimaCharlie API](#phase-7---isolation-through-the-limacharlie-api)
6. [Detection rule](#6-detection-rule)
7. [Tines workflow](#7-tines-workflow)
8. [End-to-end test results](#8-end-to-end-test-results)
9. [Troubleshooting](#9-troubleshooting)
10. [Security notes](#10-security-notes)
11. [Limitations and next steps](#11-limitations-and-next-steps)
12. [What I learned](#12-what-i-learned)
13. [Quick links to all documentation](#13-quick-links-to-all-documentation)
14. [Screenshot index](#14-screenshot-index)
15. [Credits](#15-credits)
16. [License](#16-license)

---

## 1. Overview

**Goal:** detect LaZagne on an endpoint, notify the analyst, let the analyst decide, and act on that decision automatically.

**What the pipeline does:**

1. LaZagne runs on the Windows VM.
2. The LimaCharlie sensor records a `NEW_PROCESS` event.
3. A detection and response (D&R) rule matches and creates a detection.
4. A LimaCharlie Output (detect stream) posts the detection to a Tines webhook.
5. Tines builds a review page, then sends a Slack alert and an email containing the detection details and a link to the review page.
6. The analyst opens the page and answers Yes or No to "Do you want to isolate this machine?".
7. No: Slack gets "The computer {hostname} was not isolated, please investigate".
8. Yes: Tines calls the LimaCharlie isolation API, checks the status, and posts "The computer {hostname} has been isolated. Status: true" to Slack.

**Skills practiced:** detection engineering, SOAR workflow design, human-in-the-loop response, REST API authentication, and troubleshooting with curl.

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 2. Architecture

Design flowchart (the full intended design, including the optional status-polling and failure path):

![Design flowchart](screenshots/01-workflow-diagram.png)

Data flow as built (matches the final Tines canvas in [section 7](#7-tines-workflow)):

```
LaZagne runs on the Windows VM
  -> LimaCharlie sensor sees NEW_PROCESS
  -> D&R rule matches and creates a detection
  -> LimaCharlie Output (detect stream) posts to the Tines webhook
  -> Tines "Retrieve Detections"
       -> "User Prompt" (review page with Yes/No)
            -> "Send Slack alert"   (#alerts)
            -> "Send Email"         (Gmail SMTP)
            -> "If Else" on the isolate answer
                 isolate = false -> "Send investigation alert" (Slack)
                 isolate = true  -> "Isolate sensor"            (POST)
                                 -> "Check isolation status"    (GET)
                                 -> "Send isolation confirmation" (Slack)
```

Early in the build, Slack and email ran in parallel straight off the webhook ([screenshot](screenshots/15-tines-email-step.png)). I later put the review page first so the alerts could include a "Detection Review URL" link.

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 3. Tools and environment

| Item | Value |
|---|---|
| Endpoint | Windows Server 2022 VM in VirtualBox (4 GB RAM, 50 GB disk, NAT network, Guest Additions) |
| Hostname | `addc01.varun.local` |
| Host machine | Windows host (Chrome, PowerShell) |
| EDR | LimaCharlie (free tier), Windows 64-bit sensor |
| SOAR | Tines (new prompt-driven interface, workflows) |
| Chat | Slack workspace `VARUN-SOAR-EDR`, channel `#alerts` |
| Email | Gmail through the Tines Gmail (SMTP) connector with an App Password |
| Test tool | LaZagne (credential recovery tool), run as `.\LaZagne.exe all` |
| LaZagne SHA256 | `dc06d62ee95062e714f2566c95b8edaabfd387023b1bf98a09078b84007d5268` |
| Defender | Real-time protection off on the VM so LaZagne can run |
| Firewall rules | None needed. The VM only makes outbound connections over NAT |

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 4. Repository layout

```
soar-edr-tines-limacharlie/
  README.md                         this file
  LICENSE                           MIT
  .gitignore
  detection/
    lazagne-detection-rule.yaml     final D&R rule (detect + respond)
  docs/
    setup-guide.md                  detailed phase-by-phase notes
    tines-prompts.md                every prompt used in the Tines AI box
    api-reference.md                LimaCharlie API calls and curl tests
    field-reference.md              verified webhook field paths
    troubleshooting.md              symptoms, causes, fixes
    security-notes.md               what to redact and rotate before publishing
    screenshot-index.md             all screenshots with captions
  screenshots/
    01-workflow-diagram.png ... 29-final-canvas.png
    30-varun-slack-connected.png ... 33-varun-story-notes.png
```

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 5. Walkthrough

This is the short version with the key screenshots. The full step-by-step notes are in [docs/setup-guide.md](docs/setup-guide.md).

### Phase index

| Phase | What I did | Result |
|---|---|---|
| [1. Design](#phase-1---design) | Drew the flow before building | Flowchart with Yes/No branch |
| [2. LimaCharlie and the VM](#phase-2---limacharlie-and-the-windows-vm) | VirtualBox Windows Server 2022 VM plus EDR sensor | Sensor receiving telemetry |
| [3. Detection rule](#phase-3---telemetry-and-the-detection-rule) | Ran LaZagne, wrote and tested the D&R rule | Rule matched, live detection |
| [4. Slack and webhook](#phase-4---slack-workspace-and-tines-webhook) | Workspace, `#alerts`, Tines webhook, LimaCharlie Output | Detection arrives in Tines |
| [5. Alerts](#phase-5---alerts-slack-and-email) | Custom Slack app, Gmail SMTP, exact field paths | Real data in Slack and email |
| [6. Review page](#phase-6---human-decision-page-and-branching) | Yes/No page and If Else | Both branches work |
| [7. Isolation](#phase-7---isolation-through-the-limacharlie-api) | Token-exchange connector, POST and GET to LimaCharlie | Sensor isolated, "Status: true" in Slack |
| [8. Final test](#8-end-to-end-test-results) | One Yes run and one No run | Both verified in screenshots |

### Phase 1 - Design

I drew the workflow before building anything: detection, webhook, three parallel outputs, a Yes/No decision, and the isolation branch. See the flowchart in [section 2](#2-architecture).

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation) | [Phase index](#phase-index)

### Phase 2 - LimaCharlie and the Windows VM

The video used a cloud server. I used a local VirtualBox VM instead, which needs no firewall rules because it only makes outbound connections.

1. Create a Windows Server 2022 VM (4 GB RAM, 50 GB disk, NAT, Guest Additions) and set the hostname to `addc01.varun.local`.
2. Turn off Windows Defender real-time protection so LaZagne can run.
3. Create a LimaCharlie organization and an installation key.
4. Download the Windows 64-bit EDR sensor and install it from an admin PowerShell with `-i <installation key>`.
5. Confirm the sensor appears in the Sensors list.

![Installation key](screenshots/02-installation-key.png)

![Sensor enrolled](screenshots/03-sensor-enrolled.png)

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation) | [Phase index](#phase-index)

### Phase 3 - Telemetry and the detection rule

I downloaded LaZagne, kept it past SmartScreen, and ran it on the VM:

![LaZagne running](screenshots/18-lazagne-execution.png)

The sensor Timeline shows the `NEW_PROCESS` events, including the file hash:

![Timeline events](screenshots/04-timeline-event.png)

I built the D&R rule by adapting LimaCharlie's public "Windows Process Creation" example. The detect logic and the response action:

![Detect logic](screenshots/05-detection-rule.png)

![Response action](screenshots/06-response-action.png)

I tested it with the Target Event / Test Event panel. The rule matched; the evaluation trace short-circuits after the first true condition (the file path ending in `lazagne.exe`), so the other three show as "not evaluated".

![Rule test](screenshots/07-rule-test.png)

Then I cleared old detections and ran `lazagne all` again to get a live detection:

![Live detection](screenshots/08-live-detection.png)

The full rule is in [section 6](#6-detection-rule).

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation) | [Phase index](#phase-index)

### Phase 4 - Slack workspace and Tines webhook

I created a Slack workspace and a public `#alerts` channel.

![Slack workspace](screenshots/09-slack-workspace.png)

![alerts channel](screenshots/10-slack-alerts-channel.png)

The new Tines interface has no blank canvas on the dashboard. You create a workflow by typing a prompt into the AI box. My first prompt created the webhook and tested it:

![Tines webhook created](screenshots/11-tines-webhook-created.png)

A draft webhook does not accept outside traffic, so I used "Push live" and copied the webhook URL from the action card. In LimaCharlie I added an Output (Detections, type Tines) with that URL:

![LimaCharlie output](screenshots/12-limacharlie-output.png)

After running LaZagne again, the same detection showed up in the Tines webhook Events tab:

![Webhook event in Tines](screenshots/13-tines-webhook-event.png)

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation) | [Phase index](#phase-index)

### Phase 5 - Alerts: Slack and email

**Slack.** Tines now asks for a custom Slack app instead of the pre-built one. I created an app at api.slack.com/apps, added the bot scope `chat:write`, installed it, and pasted the Bot User OAuth Token into the Tines Slack connector. The first test failed with `not_in_channel`; inviting the app to `#alerts` fixed it.

![First Slack test message](screenshots/14-slack-test-message.png)

**Email.** Gmail OAuth in Tines is in limited preview and was denied for my account, so I used the Gmail (SMTP) connector with a Google App Password (this requires 2-Step Verification). Early canvas with the email step added next to Slack:

![Email step in Tines](screenshots/15-tines-email-step.png)

The first alerts showed "Not provided" for most fields because the AI had guessed the field paths. After I sent the exact paths ([docs/field-reference.md](docs/field-reference.md)), real data came through:

![First real Slack alert](screenshots/16-slack-alert-first-real.png)

![First real email alert](screenshots/17-gmail-alert-first-real.png)

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation) | [Phase index](#phase-index)

### Phase 6 - Human decision page and branching

A Tines Page action named "User Prompt" shows the detection fields and a Yes/No question:

![User Prompt page](screenshots/19-user-prompt-page.png)

The final alerts include a "Detection Review URL" that opens this page:

![Final Slack alert](screenshots/20-slack-alert-final.png)

![Final email alert](screenshots/21-email-alert-final.png)

An If Else action checks the `isolate` answer. No sends "The computer {hostname} was not isolated, please investigate" to Slack. Yes continues to the isolation steps.

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation) | [Phase index](#phase-index)

### Phase 7 - Isolation through the LimaCharlie API

This was the hardest part. The built-in Tines LimaCharlie connector failed, so I tested the API outside Tines with curl, then built my own connector (details in [section 7](#7-tines-workflow) and [docs/api-reference.md](docs/api-reference.md)). Once both HTTP steps used the working connector, the Yes branch ran: isolate, check status, confirm in Slack.

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation) | [Phase index](#phase-index)

---

## 6. Detection rule

The same rule is saved as [detection/lazagne-detection-rule.yaml](detection/lazagne-detection-rule.yaml).

**Detect logic:** event type `NEW_PROCESS` or `EXISTING_PROCESS`, platform Windows, and any one of four conditions (all case-insensitive):

| # | Condition | Value |
|---|---|---|
| 1 | `event/FILE_PATH` ends with | `lazagne.exe` |
| 2 | `event/COMMAND_LINE` ends with | `all` |
| 3 | `event/COMMAND_LINE` contains | `lazagne` |
| 4 | `event/HASH` is | `dc06d62ee95062e714f2566c95b8edaabfd387023b1bf98a09078b84007d5268` |

**Response:** `report` action with metadata (author, description, level medium, tag `attack.credential_access`). The detection title is `Varun - HackTool - LaZagne (SOAR-EDR)`.

### MITRE ATT&CK mapping

| Tactic | Technique | How it relates |
|---|---|---|
| Credential Access (TA0006) | [T1555 - Credentials from Password Stores](https://attack.mitre.org/techniques/T1555/) | LaZagne recovers stored passwords from many applications |
| Credential Access (TA0006) | [T1555.003 - Credentials from Web Browsers](https://attack.mitre.org/techniques/T1555/003/) | One of the main LaZagne modules targets browser password stores |

The rule is tagged `attack.credential_access`. The response side of the project (isolating the host) is a containment action rather than an ATT&CK technique.

Notes on the design: the hash condition catches the exact binary even if it is renamed, while the path and command-line conditions catch other builds of LaZagne. Condition 2 alone (`all` at the end of a command line) is broad and would be a source of false positives in a real environment; the four conditions are joined with `or`, so any one of them is enough to fire, which suits a lab demo. I note this in [Limitations](#11-limitations-and-next-steps).

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 7. Tines workflow

Final canvas, left to right:

![Final Tines canvas](screenshots/29-final-canvas.png)

| Step | Type | What it does |
|---|---|---|
| Retrieve Detections | Webhook | Receives the detection posted by the LimaCharlie Output |
| User Prompt | Page | Shows the detection fields and the Yes/No "isolate" question |
| Send Slack alert | Slack | Posts the labeled detection fields and review link to `#alerts` |
| Send Email | Gmail (SMTP) | Sends the same details with `<br>` line breaks |
| If Else | Condition | Branches on the `isolate` answer |
| Send investigation alert | Slack | No branch: "The computer {hostname} was not isolated, please investigate" |
| Isolate sensor | HTTP POST | `POST https://api.limacharlie.io/v1/{sid}/isolation` |
| Check isolation status | HTTP GET | `GET https://api.limacharlie.io/v1/{sid}/isolation` |
| Send isolation confirmation | Slack | "The computer {hostname} has been isolated. Status: {status}" |

**Fields shown in the alerts:** Title, Time, Computer, Username, Source IP, File Path, Command Line, Sensor ID, Detection Link, and the Detection Review URL. The email also carries Detection ID, Process ID, and Parent Process ID. The exact paths are in [docs/field-reference.md](docs/field-reference.md).

**LimaCharlie authentication connector (`LimaCharlie JWT`).** LimaCharlie API calls need a JWT that you get by exchanging the organization ID and an API key. In Tines I created a connector with these settings:

| Setting | Value |
|---|---|
| Connector type | API key, Token exchange |
| Allowed URLs | `api.limacharlie.io` |
| Token URL | `https://jwt.limacharlie.io` |
| Body format | Form encoded |
| Body fields | `oid` and `secret` |
| Token path | `jwt` |
| Token lifetime | `3000` seconds (the JWT is valid for 1 hour) |
| Header | Bearer |
| Test URL | `GET https://api.limacharlie.io/v1/sensors/<OID>` |

The API key (`tines-final`) has only three permissions: `sensor.get` (read), `sensor.list` (read), and `sensor.task` (write). Never commit the key or the OID/secret pair; see [section 10](#10-security-notes).

All prompts I used to build this workflow are in [docs/tines-prompts.md](docs/tines-prompts.md), including a master prompt that rebuilds the same structure from scratch.

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 8. End-to-end test results

I ran two fresh detections with `.\LaZagne.exe all` on the VM, one for each branch.

**Test 1 - answer Yes (isolate)**

1. The detection arrived in Slack and email with a review link.
2. I opened the review page, selected Yes, and submitted.

![Select Yes](screenshots/22-decision-yes.png)

![Response recorded, Yes](screenshots/23-response-recorded-yes.png)

3. Slack received the isolation confirmation:

![Isolation confirmation](screenshots/24-slack-isolation-confirmation.png)

4. LimaCharlie shows the sensor as Isolated, and the button now reads Rejoin Network:

![Sensor isolated](screenshots/25-sensor-isolated.png)

After the test I rejoined the VM to the network with the Rejoin Network button (or `DELETE /v1/{sid}/isolation`, see [docs/api-reference.md](docs/api-reference.md)).

**Test 2 - answer No (investigate)**

![Select No](screenshots/26-decision-no.png)

![Response recorded, No](screenshots/27-response-recorded-no.png)

![Not isolated message](screenshots/28-slack-not-isolated.png)

| Test | Answer | Result in Slack | Result in LimaCharlie |
|---|---|---|---|
| 1 | Yes | "has been isolated. Status: true" | Sensor Isolated |
| 2 | No | "was not isolated, please investigate" | Sensor unchanged |

The review page records only the first answer for each detection. To test again, trigger a new detection.

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 9. Troubleshooting

The short table is below; the detailed version with the full LimaCharlie authentication story is in [docs/troubleshooting.md](docs/troubleshooting.md).

| Symptom | Cause | Fix |
|---|---|---|
| Alerts show "Not provided" | Wrong field paths guessed by the AI | Use the paths in [docs/field-reference.md](docs/field-reference.md) |
| Slack error `not_in_channel` | Bot is not in the channel | Run `/invite @VARUN-SOAR-EDR` in `#alerts` |
| Gmail connector denied | Gmail OAuth is limited preview | Use Gmail (SMTP) with an App Password (needs 2-Step Verification) |
| Webhook gets events but Slack, Email, and the page do not run | Downstream steps do not auto-run on new events while a draft is being edited | Push live, or use the replay prompts in [docs/tines-prompts.md](docs/tines-prompts.md) |
| Review page shows "Response recorded" immediately | Only the first answer per detection is accepted | Trigger a new detection |
| Isolate step returns 401 | Invalid or deleted API key, or the step is still attached to the old connector | Test the key with curl, then attach `LimaCharlie JWT` to both HTTP steps |
| JWT call returns `unknown api key` | The key does not exist in LimaCharlie | Create a new key with the three sensor permissions |
| JWT call returns `missing secret` | Secret field blank or not saved | Re-enter the key and confirm it saved after reopening the connector |
| Connector test shows 404 | The generic test hits an unrelated URL | Test the real action or the sensors endpoint instead |
| Email has no line breaks | Body is HTML | Use `<br>` |
| Trailing dot in a variable path | Invalid path | Remove the dot |

**About the 401 errors:** during troubleshooting I suspected a Tines platform limitation. That turned out to be wrong. The verified causes were an invalid or deleted API key, plus the two HTTP steps still being attached to the old connector.

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 10. Security notes

Credentials are kept out of this repository, and the checklist I followed before publishing is in [docs/security-notes.md](docs/security-notes.md). In short:

- The LimaCharlie API key used by Tines has only three permissions (`sensor.get`, `sensor.list`, `sensor.task`), and any key or App Password that was ever pasted into a chat should be revoked and replaced.
- The Tines webhook URL (it contains an `external_id` token) and the review page URL are treated as secrets and are blacked out in screenshots.
- The public IP, the Gmail address, and secret-bearing headers are blacked out in screenshots. I used solid black boxes rather than blur because blurred text can sometimes be recovered.
- No keys, tokens, or webhook URLs are stored anywhere in this repository.

The organization ID and sensor ID remain visible in some screenshots. They are identifiers, not credentials, but you may want to hide them in your own copy.

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 11. Limitations and next steps

- **Raw timestamps.** The `Time` field is an epoch value in milliseconds (for example `1791360279185`). Converting it to a readable date inside Tines is a possible improvement.
- **Broad rule condition.** The `command line ends with all` condition would cause false positives outside a lab. A production rule should rely on the hash, the file path, and parent-process context.
- **Single approver, no authentication on the page.** Anyone with the review link can answer. A real deployment should put the page behind authentication and log who approved.
- **Hash-based detection is easy to evade.** A recompiled or packed LaZagne has a different hash. Behavioral detections (access to browser credential stores, LSASS access) would be stronger.
- **No automatic rejoin.** Un-isolating is manual (button or API call).
- **Ideas:** a timeout on the review page with a default action, a ticket created in a case-management tool, threat-intel enrichment of the hash before the question is asked, and a second rule for another tool such as Mimikatz.

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 12. What I learned

- How a D&R rule is structured in LimaCharlie, how to adapt a public example, and how to read the evaluation trace in the test panel.
- Why AI-generated field mappings must be verified against the real payload. The first alerts were mostly empty because paths and capitalization were guessed.
- How to debug API authentication from outside the tool: testing the key with curl (JWT exchange, list sensors, isolate, rejoin) separated "my key is bad" from "my connector is wrong".
- That the real fault is often simpler than the theory. I blamed the platform first, but the causes were a deleted key and steps still pointing at the old connector.
- How to design a human-in-the-loop response: the automation gathers context and acts, but a person approves the disruptive step.
- How SOAR connectors differ from the tutorial over time (custom Slack apps, SMTP instead of Gmail OAuth, prompt-driven workflows), and how to adapt instead of copying.

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 13. Quick links to all documentation

| Document | What is in it |
|---|---|
| [docs/setup-guide.md](docs/setup-guide.md) | Detailed phase-by-phase build notes |
| [docs/tines-prompts.md](docs/tines-prompts.md) | Every Tines prompt, replay prompts, health check, master rebuild prompt |
| [docs/api-reference.md](docs/api-reference.md) | LimaCharlie endpoints and curl tests |
| [docs/field-reference.md](docs/field-reference.md) | Verified webhook field paths |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Symptoms, causes, and fixes |
| [docs/security-notes.md](docs/security-notes.md) | Redaction and rotation checklist |
| [docs/screenshot-index.md](docs/screenshot-index.md) | All screenshots with captions |
| [detection/lazagne-detection-rule.yaml](detection/lazagne-detection-rule.yaml) | The detection rule |

[Back to top](#soar--edr-automation-lazagne-detection-human-approval-and-automatic-isolation)

---

## 14. Screenshot index

The complete list with captions is in [docs/screenshot-index.md](docs/screenshot-index.md). Folder: [screenshots/](screenshots/).

---

## 15. Credits

- The MyDFIR "SOAR EDR project" YouTube series is the base project this work follows and adapts.
- LimaCharlie documentation and its public example rules.
- LaZagne by Alessandro Zanni (AlessandroZ) is used here only as a test tool on a private VM.

---

## 16. License

MIT. See [LICENSE](LICENSE).

## Author

**B VARUN**

🔗 **GitHub:** https://github.com/VARUN-B18  
🔗 **LinkedIn:** https://www.linkedin.com/in/varunb-/
