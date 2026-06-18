---
title: UniFi Content Filter via API
type: docs
date: 2026-03-23
description: How to apply a DNS blocklist to UniFi's content filter using the API
---

{{< callout type="warning" >}}
**Heads up:** I did not enjoy managing DNS blocks this way, it got too tedious. AdGuard handles this much better, so I moved my internal DNS back to AdGuard and now I prefer to let it handle these blocks. I'm keeping this article as a reference for how to use UniFi's content filter API, but it is no longer the path I use.
{{< /callout >}}

UniFi OS 4.x introduced a content filtering feature (Cybersecure menu) that supports custom domain block lists per network. The API endpoint is not officially documented but works with the standard API key.

## Requirements

- UniFi OS 4.x (tested on UniFi Express 7, firmware 4.4.x)
- API key with network access (`Settings → Control Plane → API Keys`)
- The content filter profile ID (see below)

## Finding your content filter profile ID

The easiest way is to open the UniFi UI, navigate to **Cybersecure → Content Filtering**, enable a rule, and intercept the request in browser devtools (Network tab). Look for a PUT request to:

```
/proxy/network/v2/api/site/default/content-filtering/<profile-id>
```

Copy the profile ID from the URL.

Alternatively, you can try GET on the collection, though this endpoint may require cookie auth depending on your firmware version:

```bash
curl -sk https://<unifi-ip>/proxy/network/v2/api/site/default/content-filtering \
  -H "X-API-KEY: <your-api-key>"
```

## Payload structure

```json
{
  "_id": "<profile-id>",
  "enabled": true,
  "name": "Default",
  "categories": ["ADVERTISEMENT"],
  "client_macs": [],
  "network_ids": ["<network-id>"],
  "allow_list": [],
  "block_list": ["example.com", "ads.example.com"],
  "safe_search": [],
  "schedule": { "mode": "ALWAYS" }
}
```

**Fields:**
- `categories`: built-in DNS Shield categories (e.g. `ADVERTISEMENT`, `ADULT`, `GAMBLING`, `MALWARE`)
- `block_list`: array of domains to block (custom, applied on top of categories)
- `allow_list`: domains to always allow, overriding blocks
- `network_ids`: which networks this profile applies to
- `client_macs`: limit to specific devices (empty = all devices on the network)
- `schedule.mode`: `ALWAYS` or time-based

## Downloading and parsing a blocklist

This example uses [Peter Lowe's Ad/Tracker list](https://pgl.yoyo.org/adservers/) (~3.5k domains, hosts format):

```bash
curl -s "https://pgl.yoyo.org/adservers/serverlist.php?hostformat=hosts&showintro=0&mimetype=plaintext" \
  -o /tmp/blocklist.txt
```

Parse to extract plain domains:

```python
domains = []
with open('/tmp/blocklist.txt') as f:
    for line in f:
        line = line.strip()
        if line.startswith('#') or not line:
            continue
        # format: "127.0.0.1 domain.com"
        parts = line.split()
        if len(parts) >= 2 and parts[1] != 'localhost':
            domains.append(parts[1])

print(f"Parsed {len(domains)} domains")
```

Other lists in hosts format that work with the same parser:
- [Steven Black's unified hosts](https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts)
- [Dan Pollock's hosts](https://someonewhocares.org/hosts/hosts)

## Applying to UniFi

```python
import json
import subprocess

UNIFI_IP = "<unifi-ip>"
API_KEY = "<your-api-key>"
PROFILE_ID = "<your-profile-id>"
NETWORK_ID = "<your-network-id>"

payload = {
    "_id": PROFILE_ID,
    "enabled": True,
    "name": "Default",
    "categories": ["ADVERTISEMENT"],
    "client_macs": [],
    "network_ids": [NETWORK_ID],
    "allow_list": [],
    "block_list": domains,
    "safe_search": [],
    "schedule": {"mode": "ALWAYS"}
}

with open('/tmp/payload.json', 'w') as f:
    json.dump(payload, f)

result = subprocess.run([
    "curl", "-sk", "-X", "PUT",
    f"https://{UNIFI_IP}/proxy/network/v2/api/site/default/content-filtering/{PROFILE_ID}",
    "-H", "content-type: application/json",
    "-H", f"X-API-KEY: {API_KEY}",
    "-d", "@/tmp/payload.json"
], capture_output=True, text=True)

data = json.loads(result.stdout)
print(f"Applied {len(data.get('block_list', []))} domains")
```

Or as a one-liner with bash:

```bash
UNIFI_IP="<unifi-ip>"
API_KEY="<your-api-key>"
PROFILE_ID="<your-profile-id>"

curl -sk -X PUT "https://$UNIFI_IP/proxy/network/v2/api/site/default/content-filtering/$PROFILE_ID" \
  -H "content-type: application/json" \
  -H "X-API-KEY: $API_KEY" \
  -d @/tmp/payload.json \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('Applied:', len(d.get('block_list',[])))"
```

## Notes

- The API accepts the full list in a single PUT, no pagination needed
- Tested with ~3,500 domains (~70KB payload), worked without issues
- Larger lists (50k+) may hit payload size limits, test before assuming they work
- The content filter operates at the network level, before DNS resolvers like AdGuard
- UniFi does not expose logs or stats for content filter blocks via API key, only through the UI or authenticated browser sessions
- Built-in `categories` and custom `block_list` are additive, both apply simultaneously
