# Security notes

[Back to README](../README.md) | [API reference](api-reference.md)

A checklist for anyone publishing a project like this one.

## Rotate or revoke before publishing

| Item | Why | Action |
|---|---|---|
| Gmail App Password | If it was ever pasted into a chat or file, treat it as exposed | Revoke at myaccount.google.com/apppasswords, create a new one, update the Tines Gmail (SMTP) connector |
| LimaCharlie API key | Keys pasted anywhere should be considered exposed | Delete old keys. Delete the working key (`tines-final`) when the project is finished, or keep it private |
| LimaCharlie JWT | Expires after 1 hour, so a pasted JWT is dead | Nothing to do, but do not rely on this |
| Tines webhook URL | Contains an `external_id` token. Anyone with it can send fake detections | Treat as a secret. Push a new webhook and update the LimaCharlie Output if it was shown |
| Review page URL | Anyone with the link can answer the isolate question | Treat as a secret. Do not leave it visible in screenshots |
| Slack bot token (`xoxb-...`) | Lets anyone post as the bot | Never commit or screenshot it. Reinstall the app to rotate it |

## Redaction rules for screenshots

Use solid black boxes. Blur and pixelate can sometimes be reversed.

Redact:

- Public (external) IP addresses, including the `ext_ip` field in detection JSON
- Your Gmail address and any email address
- The Tines webhook URL, the `external_id` value, and the host name (it can contain your account handle)
- The `lc-signature` header
- The review page URL (`/user-prompt?id=...`)
- API keys, bot tokens, and App Passwords

Optional to redact: organization ID, sensor ID, installer ID, MAC address. These are identifiers, not credentials, but they reveal your setup.

## Repository hygiene

- Use placeholders such as `<OID>`, `<SID>`, and `<your-api-key-secret>` in every doc.
- The `.gitignore` blocks common secret file names, but it is a safety net, not a guarantee. Review `git status` and `git diff --staged` before each commit.
- Run `git log -p | grep -i -E "xoxb|secret|password|external_id"` before pushing, to catch anything that slipped in.
- If a secret is ever committed, rotating it is the fix. Rewriting history does not undo exposure.

## Lab safety

- LaZagne was run only on a personal test VM.
- Windows Defender real-time protection was turned off on that VM only, and the VM uses NAT.
- The isolation step is intentionally gated behind a human decision.
