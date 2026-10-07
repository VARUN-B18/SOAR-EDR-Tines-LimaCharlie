# Field reference

[Back to README](../README.md) | [Setup guide](setup-guide.md) | [Prompts](tines-prompts.md)

These are the verified, case-sensitive paths from the real detection JSON received by the Tines webhook. Paths are relative to the webhook body.

| Label | Path | Example value |
|---|---|---|
| Title | `cat` (top level, not under `detect`) | Varun - HackTool - LaZagne (SOAR-EDR) |
| Time | `detect.routing.event_time` | <epoch-ms> (epoch milliseconds) |
| Computer | `detect.routing.hostname` | addc01.varun.local |
| Username | `detect.event.USER_NAME` | VARUN-SOAR-EDR\Administrator |
| Source IP | `detect.routing.int_ip` | <internal-ip> |
| File Path | `detect.event.FILE_PATH` | C:\Users\Administrator\Downloads\LaZagne.exe |
| Command Line | `detect.event.COMMAND_LINE` | "C:\Users\Administrator\Downloads\LaZagne.exe" all |
| Sensor ID | `detect.routing.sid` | <sensor-id> |
| Detection Link | `link` (top level, not under `detect`) | https://app.limacharlie.io/orgs/.../timeline?... |
| Hash (extra) | `detect.event.HASH` | dc06d62e... |

## Notes

- `USER_NAME`, `FILE_PATH`, `COMMAND_LINE`, and `HASH` are uppercase.
- `cat` and `link` are top-level fields. They are not under `detect`.
- The first version of my alerts showed "Not provided" for almost every field. The AI had guessed lowercase and wrongly nested paths such as `detect.cat`, `detect.link`, `event.user`, and `internal_ip`. Sending the exact paths in a prompt fixed it (prompt F in [tines-prompts.md](tines-prompts.md)).
- `event_time` is epoch milliseconds. Divide by 1000 for a normal Unix timestamp. On a Unix-like system: `date -r $((<epoch-ms>/1000))`.
- The raw webhook body also carries `detect.event.PARENT` (the parent process with its own command line, hash, and process ID), `detect.routing.ext_ip` (public IP, treat as sensitive), and `detect.routing.oid`.

## Screenshots

- [Webhook event in Tines](../screenshots/13-tines-webhook-event.png)
- [Final Slack alert](../screenshots/20-slack-alert-final.png)
