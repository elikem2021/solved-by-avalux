[![Maintained by Avalux](https://img.shields.io/badge/Maintained%20by-Avalux.io-3b82f6?style=flat-square)](https://avalux.io)

# Recipe: Route Inbound Leads to the Right Rep by Postal Code

> The classic SMB pain point: leads land in a shared inbox, the wrong person picks them up two hours later, the right person never sees them, and the customer goes to a competitor. The fix is a tiny postal-code → rep lookup that fires the moment a lead lands.

## The problem

You have 4 reps, each owning a territory. Leads come in via website forms, phone, cold-email replies, Facebook ads. Right now they all go to one inbox or one CRM stage and someone has to manually triage and assign.

By the time triage happens, the lead has cooled. Studies consistently show conversion rates drop 80% after the first 5 minutes of lead arrival. You're leaving money on the floor every day.

## Who has this problem

- Service businesses with territory-based reps (real estate, insurance, home improvement, freight brokers)
- Multi-location franchises routing inbound leads to the nearest location
- B2B sales teams with named-account territories
- Anyone whose closing logic depends on geographic match

## When this fix makes sense

- Your lead sources can emit a webhook or push to a known endpoint (most can)
- You have at least 2 reps and a clear territory map
- Your average lead value justifies investing 1-2 days in setup

## When it doesn't make sense

- You have 1 rep doing everything (don't optimize what doesn't need optimizing)
- Territories are fuzzy or change weekly (the lookup table churn is worse than manual)

## The free DIY path

Three moving parts:

1. **Inbound webhook** — receives lead from form/CRM/cold-email-reply parser
2. **Lookup** — postal code (or first 3 chars in Canada — the FSA) maps to a rep
3. **Route** — post to the right rep's Slack DM (or text them, or push to their CRM stage)

Below is a minimum-viable Python webhook in ~50 lines, using Canadian FSA codes as the example. US ZIP codes work the same way (use the first 3 of a 5-digit ZIP as the territory key).

```python
# lead_router.py
from flask import Flask, request, abort
import os, requests, json, re

app = Flask(__name__)

WEBHOOK_SECRET = os.environ["WEBHOOK_SECRET"]
SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK_URL"]

# Map FSA (first 3 chars of Canadian postal code) → rep slug
TERRITORIES = {
    # Mississauga / Brampton (Alex)
    "L4T": "alex", "L4V": "alex", "L4W": "alex", "L4X": "alex",
    "L5A": "alex", "L5B": "alex", "L5C": "alex", "L5G": "alex",
    "L5H": "alex", "L5J": "alex", "L5K": "alex", "L5L": "alex",
    "L5M": "alex", "L5N": "alex", "L5R": "alex", "L5T": "alex",
    "L5V": "alex", "L5W": "alex",
    # Toronto + Etobicoke (Sam)
    "M1A": "sam", "M2A": "sam", "M3A": "sam", "M4A": "sam",
    "M5A": "sam", "M6A": "sam", "M7A": "sam", "M8X": "sam",
    "M9A": "sam", "M9B": "sam", "M9C": "sam", "M9V": "sam",
    # Hamilton (Jamie)
    "L8E": "jamie", "L8G": "jamie", "L8H": "jamie", "L8J": "jamie",
    "L8K": "jamie", "L8L": "jamie", "L8M": "jamie", "L8N": "jamie",
    "L8P": "jamie", "L8R": "jamie", "L8S": "jamie", "L8T": "jamie",
    "L8V": "jamie", "L8W": "jamie",
    # Default catch-all (rotates)
}

REPS = {
    "alex": {"slack_user_id": "U01ALEX", "name": "Alex"},
    "sam": {"slack_user_id": "U01SAM", "name": "Sam"},
    "jamie": {"slack_user_id": "U01JAMIE", "name": "Jamie"},
}


def normalize_postal(p):
    """Canadian: 'L5B 3Y6' → 'L5B'. US: '78704-1234' → '787' (use 3-digit ZIP)."""
    p = (p or "").upper().strip()
    if re.match(r"^[A-Z]\d[A-Z]", p):
        return p[:3]  # Canadian FSA
    if re.match(r"^\d{5}", p):
        return p[:3]  # US 3-digit ZIP
    return None


@app.post("/webhook/lead")
def on_lead():
    if request.headers.get("X-Webhook-Secret") != WEBHOOK_SECRET:
        abort(401)
    lead = request.json
    fsa = normalize_postal(lead.get("postal_code"))
    rep_slug = TERRITORIES.get(fsa, "unassigned")
    rep = REPS.get(rep_slug)

    if rep:
        msg = (
            f"<@{rep['slack_user_id']}> 🔥 New lead in your territory\n"
            f"*Name:* {lead.get('name', '?')}\n"
            f"*Phone:* {lead.get('phone', '?')}\n"
            f"*Email:* {lead.get('email', '?')}\n"
            f"*Postal:* {lead.get('postal_code', '?')} (FSA: {fsa})\n"
            f"*Source:* {lead.get('source', '?')}\n"
            f"*Message:* {lead.get('message', '')[:500]}\n"
            f"*SLA: 5 minutes.*"
        )
    else:
        msg = (
            f"🟡 Unassigned lead (no territory match)\n"
            f"*Postal:* {lead.get('postal_code', '?')}\n"
            f"*Lead:* {lead.get('name', '?')} / {lead.get('phone', '?')}\n"
            f"Manual triage required."
        )

    requests.post(SLACK_WEBHOOK, json={"text": msg}, timeout=10)
    return {"routed_to": rep_slug}, 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

Point every lead source's webhook (Webflow form, Zapier/n8n, Apollo reply parser, cold-email reply forwarder) at this endpoint.

## Common gotchas

1. **Postal code normalization.** Customers type postal codes in every imaginable format. Strip spaces, uppercase, validate. Treat invalid as `unassigned` rather than crashing.
2. **PO boxes.** A PO box postal code doesn't tell you where the customer lives. For high-value leads, fall back to phone area code or IP geolocation.
3. **Round-robin for unassigned.** Don't let "unassigned" become a graveyard. Either rotate it across reps or assign it to a sales manager whose only job is triage.
4. **Cross-territory leads.** Sometimes a lead from one territory wants service in another (snowbirds, multi-property owners). Build in a "manual reassign" Slack button so reps can hand off.
5. **Lead source attribution.** Always log the source. After 3 months, you'll know which source has the worst territory-fit (probably broad-targeted ads) and can re-tune spend.

## Variations

- **By city + service type.** If you offer different services in different cities, the lookup becomes a 2D table (city × service_type → rep).
- **Round-robin within territory.** If multiple reps cover the same area, rotate within the territory.
- **Latency-aware routing.** If rep A hasn't responded to their last 3 leads within 10 min, auto-route to rep B.
- **CRM integration.** Instead of Slack, push the lead into HubSpot/Pipedrive/GoHighLevel with the rep already assigned.

## Production version

[Avalux](https://avalux.io) builds the full lead-routing stack — multi-territory rules engine, latency-aware fallback, CRM integration (HubSpot/Pipedrive/GoHighLevel/Salesforce), response-time analytics dashboard, and reply-bot for after-hours leads. Pricing starts at $5,000. See [avalux.io/contact.html](https://avalux.io/contact.html) or email [eli@avalux.io](mailto:eli@avalux.io).

## License

MIT — part of the [solved-by-avalux](https://github.com/elikem2021/solved-by-avalux) cookbook.
