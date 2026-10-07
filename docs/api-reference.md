# LimaCharlie API reference and curl tests

[Back to README](../README.md) | [Troubleshooting](troubleshooting.md) | [Security notes](security-notes.md)

Test the API from a terminal before wiring it into Tines. This is how I found out my key was invalid: the same call that failed inside Tines also failed in curl, which proved the problem was the key and not the connector.

Use placeholders in everything you share. Never paste a real key, secret, or JWT into a chat, issue, or commit.

## Endpoints used

| Purpose | Method and URL |
|---|---|
| Get a JWT | `POST https://jwt.limacharlie.io` with form fields `oid` and `secret`. Returns `{"jwt": "..."}`, valid for 1 hour |
| List sensors | `GET https://api.limacharlie.io/v1/sensors/<OID>` |
| Isolate a sensor | `POST https://api.limacharlie.io/v1/<SID>/isolation` (success returns `{}`) |
| Check isolation | `GET https://api.limacharlie.io/v1/<SID>/isolation` (returns `is_isolated` and `should_isolate`) |
| Rejoin network | `DELETE https://api.limacharlie.io/v1/<SID>/isolation` |

## API key permissions

Create the key under Access Management > REST API with the minimum permissions:

| Permission | Level |
|---|---|
| `sensor.get` | Read |
| `sensor.list` | Read |
| `sensor.task` | Write |

## Step-by-step curl tests (macOS or Linux)

```bash
export OID="<your-organization-id>"
export KEY="<your-api-key-secret>"
export SID="<your-sensor-id>"

# 1. Exchange the key for a JWT
curl -s -X POST https://jwt.limacharlie.io \
  -d "oid=$OID" \
  --data-urlencode "secret=$KEY"

# Copy the value of "jwt" from the response
export JWT="<paste-jwt-here>"

# 2. List sensors (proves read access)
curl -s -H "Authorization: Bearer $JWT" \
  "https://api.limacharlie.io/v1/sensors/$OID"

# 3. Check isolation status (read-only)
curl -s -H "Authorization: Bearer $JWT" \
  "https://api.limacharlie.io/v1/$SID/isolation"

# 4. Isolate the sensor
curl -s -X POST -H "Authorization: Bearer $JWT" \
  "https://api.limacharlie.io/v1/$SID/isolation"

# 5. Rejoin the network
curl -s -X DELETE -H "Authorization: Bearer $JWT" \
  "https://api.limacharlie.io/v1/$SID/isolation"
```

When you finish, clear the variables:

```bash
unset OID KEY SID JWT
```

## What the responses meant

| Response | Meaning |
|---|---|
| JWT returned | OID and key are valid; the JWT's permissions match what you gave the key |
| `unknown api key` | The key does not exist in LimaCharlie (deleted or never created) |
| `missing secret` | The secret field was blank or not sent |
| HTTP 401 | Bad or expired JWT, or the key lacks the permission |
| `{}` from the POST | Isolation was accepted |
| `is_isolated: true` in the GET | The sensor is isolated |

The GET returns both `is_isolated` and `should_isolate`. The design flowchart checks `should_isolate`. Look at a real response in your own environment before deciding which field your confirmation message should depend on.

## Tines connector settings

The connector that finally worked (named `LimaCharlie JWT`):

| Setting | Value |
|---|---|
| Type | API key, Token exchange |
| Allowed URLs | `api.limacharlie.io` |
| Token URL | `https://jwt.limacharlie.io` |
| Body format | Form encoded |
| Body | `oid` and `secret` |
| Token path | `jwt` |
| Token lifetime | `3000` |
| Header | Bearer |
| Test URL | `GET https://api.limacharlie.io/v1/sensors/<OID>` |

The connector status turned Active. A read-only GET to the isolation endpoint through it returned HTTP 200 with `is_isolated: false`.

## Screenshots

- [Sensor isolated in LimaCharlie](../screenshots/25-sensor-isolated.png)
- [Slack isolation confirmation](../screenshots/24-slack-isolation-confirmation.png)
