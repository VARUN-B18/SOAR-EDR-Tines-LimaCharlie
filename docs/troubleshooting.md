# Troubleshooting

[Back to README](../README.md) | [API reference](api-reference.md) | [Prompts](tines-prompts.md)

## Contents

- [Quick table](#quick-table)
- [Alerts show "Not provided"](#alerts-show-not-provided)
- [Slack: not_in_channel](#slack-not_in_channel)
- [Gmail connector denied](#gmail-connector-denied)
- [Downstream steps do not run on new events](#downstream-steps-do-not-run-on-new-events)
- [Review page keeps saying Response recorded](#review-page-keeps-saying-response-recorded)
- [LimaCharlie authentication: the long one](#limacharlie-authentication-the-long-one)

## Quick table

| Symptom | Cause | Fix |
|---|---|---|
| Alerts show "Not provided" | Wrong field paths | Use [field-reference.md](field-reference.md) |
| Slack error `not_in_channel` | Bot not in the channel | `/invite @VARUN-SOAR-EDR` in `#alerts` |
| Gmail connector denied | Gmail OAuth is limited preview | Gmail (SMTP) with an App Password; needs 2-Step Verification |
| New detections reach the webhook but Slack, Email, and the page do not run | Downstream steps do not auto-run on new events while editing a draft | Use the replay prompts (J) in [tines-prompts.md](tines-prompts.md) |
| Review page keeps showing "Response recorded" | Only the first answer per detection is accepted | Trigger a new detection (`lazagne all`) |
| Isolate returns 401 | Invalid or deleted API key, or step still on the old connector | Test the key with curl, then attach `LimaCharlie JWT` to both steps |
| JWT call returns `unknown api key` | Key does not exist in LimaCharlie | Create a new key with the three sensor permissions |
| JWT call returns `missing secret` | Secret blank or not sent | Re-enter the key and confirm it saved after reopening the connector |
| Connector test shows 404 | Generic connector test hits an unrelated URL | Test the real action or the sensors endpoint instead |
| Trailing dot in a variable path | Invalid path | Remove the dot |
| Email shows no line breaks | Email body is HTML | Use `<br>` |

## Alerts show "Not provided"

The first Slack and email messages showed "Not provided" for almost every field. The AI had guessed field names (lowercase, wrongly nested). The real payload uses uppercase names such as `USER_NAME`, and two fields (`cat` and `link`) sit at the top level, not under `detect`.

Fix: send the exact paths in one prompt (prompt F). See [field-reference.md](field-reference.md). Compare the working result in [the first real Slack alert](../screenshots/16-slack-alert-first-real.png).

## Slack: not_in_channel

The first test message failed with `not_in_channel`. A Slack bot can only post to channels it belongs to. Invite it from inside the channel:

```
/invite @VARUN-SOAR-EDR
```

After that the test message went through ([screenshot](../screenshots/14-slack-test-message.png)). The same screenshot shows Slack announcing that the app was added to `#alerts`.

## Gmail connector denied

Gmail OAuth in Tines is in limited preview and my request was denied. Use the Gmail (SMTP) option instead:

1. Turn on Google 2-Step Verification.
2. Create an App Password at myaccount.google.com/apppasswords.
3. Enter your Gmail address and the App Password in the Tines Gmail (SMTP) connector.
4. Send a test email. Use `<br>` for line breaks because the body is HTML.

If an App Password ever ends up in a chat, a commit, or a screenshot, revoke it and create a new one.

## Downstream steps do not run on new events

While the workflow was a draft that I was editing, new webhook events appeared in the Events tab but Slack, Email, and the review page did not run. Push the workflow live, or use the replay prompts (prompt J) to run the steps against the most recent event.

## Review page keeps saying Response recorded

The page accepts only the first answer for each detection, and later visits or duplicate submits do not trigger the decision actions. This is by design (the flowchart shows it as "Page views and duplicate submissions do not trigger decision actions"). To test again, run `.\LaZagne.exe all` to create a fresh detection and use the new review link.

## LimaCharlie authentication: the long one

**Symptoms, in order:**

1. The built-in Tines LimaCharlie connector showed Error (404 on its own test).
2. The Isolate step returned 401.
3. A hand-built Token exchange connector returned 400 `missing secret`.

**Things that did not fix it:** switching connector types, renaming body parameters, a Raw Key header connector, and changing the allowed URLs.

**What actually solved it:**

1. I tested the key outside Tines with curl. The first result was `unknown api key`, and LimaCharlie showed "No API keys created yet", so the earlier keys had been deleted or revoked.
2. I created a new key (`tines-final`) with `sensor.get` (Read), `sensor.list` (Read), and `sensor.task` (Write).
3. Curl test 1: the JWT exchange returned a JWT whose permissions matched the key.
4. Curl test 2: listing sensors with the JWT worked. Curl test 3: `POST /v1/<SID>/isolation` returned `{}` and the VM was isolated. `DELETE` rejoined it.
5. I rebuilt the Tines connector (`LimaCharlie JWT`, Token exchange) with the settings in [api-reference.md](api-reference.md). Its status turned Active.
6. A read-only GET to the isolation endpoint through the connector returned HTTP 200 with `is_isolated: false`.
7. The Isolate sensor and Check isolation status steps were still attached to the old connector, which is why the 401 kept appearing. After switching both to `LimaCharlie JWT` (prompt H), the Yes branch ran: isolation succeeded, the status check returned true, and Slack received the confirmation.

**The lesson:** I first suspected a Tines platform limitation. That was a guess and it was wrong. The verified root causes were an invalid or deleted API key, and steps still attached to the old connector. Testing the key with curl first would have saved hours.

See the final result: [Slack confirmation](../screenshots/24-slack-isolation-confirmation.png) and [isolated sensor](../screenshots/25-sensor-isolated.png).
