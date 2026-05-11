[![Maintained by Avalux](https://img.shields.io/badge/Maintained%20by-Avalux.io-3b82f6?style=flat-square)](https://avalux.io)

# Recipe: Auto-Text Customer When Tech Is On The Way

> The classic field-service problem: customer doesn't know when the technician will arrive, calls the office to ask, dispatcher has to chase the tech to find out, customer is annoyed by the time the tech arrives. The fix is a single automated SMS the moment dispatch fires the "on the way" event.

## The problem

A homeowner books an HVAC, plumbing, electrical, or locksmith service. The window is "between 1pm and 5pm." It's 2:45pm and they have no idea if the tech is coming. They call the office, the office calls the tech, the tech is en route but can't pick up, the office calls back the customer, the customer is already frustrated.

Every minute the dispatcher spends on this is a minute they can't be booking new jobs. And the customer experience is bad even when the service itself is fine.

## Who has this problem

- HVAC, plumbing, electrical, and other home-services businesses
- Locksmiths, mobile mechanics, junk removal, pool cleaning, pest control
- Any business where a tech goes to a customer location and the customer wants to know when

## When this fix makes sense

- You use a field-service CRM that fires a webhook on dispatch (ServiceTitan, Jobber, Housecall Pro, Workiz, etc.)
- You have a Twilio account, or any SMS API (Plivo, MessageBird, AWS SNS — all fine)
- Volume is at least 5-10 dispatches per day — below that, it's not worth automating

## When it doesn't make sense

- You don't have a CRM that emits dispatch events (no clean signal to trigger on)
- You bill against minimum-fee jobs and the customer doesn't really need notification
- Your techs are 100% same-day, on a fixed loop, no ETAs to share

## The free DIY path

Three moving parts:

1. **The trigger** — your CRM fires a webhook to a URL you control when a job is dispatched
2. **The lookup** — your handler grabs the customer's phone, tech's name + photo, and current ETA from your CRM's API
3. **The send** — Twilio sends a templated SMS

Below is a minimum-viable Python webhook handler in ~40 lines. Drop it on a $5 VPS behind any reverse proxy (Caddy, Nginx) or run it on n8n.

```python
# tech_on_the_way.py
# Receives a dispatch webhook from your field-service CRM, looks up tech + customer,
# and sends a templated SMS via Twilio.

from flask import Flask, request, abort
import os, requests
from twilio.rest import Client

app = Flask(__name__)

CRM_API_KEY = os.environ["CRM_API_KEY"]
CRM_BASE_URL = os.environ["CRM_BASE_URL"]  # e.g. https://api.servicetitan.io/jpm/v2
TWILIO_SID = os.environ["TWILIO_SID"]
TWILIO_TOKEN = os.environ["TWILIO_TOKEN"]
TWILIO_FROM = os.environ["TWILIO_FROM"]
WEBHOOK_SECRET = os.environ["WEBHOOK_SECRET"]  # CRM should HMAC-sign requests

tw = Client(TWILIO_SID, TWILIO_TOKEN)


@app.post("/webhook/dispatch")
def on_dispatch():
    # 1. Verify signature (implementation depends on your CRM)
    if request.headers.get("X-Webhook-Secret") != WEBHOOK_SECRET:
        abort(401)

    payload = request.json
    job_id = payload["job_id"]

    # 2. Pull job details (customer phone, tech, ETA window)
    job = requests.get(
        f"{CRM_BASE_URL}/jobs/{job_id}",
        headers={"Authorization": f"Bearer {CRM_API_KEY}"},
        timeout=10,
    ).json()

    customer_phone = job["customer"]["phone"]
    customer_name = job["customer"]["first_name"]
    tech_name = job["technician"]["first_name"]
    tech_photo_url = job["technician"].get("photo_url")
    eta = job["dispatch"]["eta_minutes"]  # e.g. "30"

    # 3. Compose + send the SMS
    body = (
        f"Hi {customer_name}, {tech_name} from [YourCompany] is on the way and "
        f"should arrive in about {eta} minutes."
    )
    if tech_photo_url:
        body += f" Quick photo so you know who to expect: {tech_photo_url}"
    body += " Reply STOP to opt out."

    tw.messages.create(to=customer_phone, from_=TWILIO_FROM, body=body)
    return {"sent": True}, 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

Deploy:

```bash
pip install flask requests twilio
gunicorn tech_on_the_way:app --bind 0.0.0.0:8080
```

Point your CRM's dispatch webhook at `https://your-domain.com/webhook/dispatch` with the secret in the header.

## Common gotchas

1. **Compliance.** In Canada (CASL) and the US (TCPA), you need consent before texting customers. Most field-service CRMs already capture this at booking. Make sure your booking form has a checkbox or recorded consent.
2. **STOP handling.** Twilio handles STOP/UNSUBSCRIBE automatically and will reject sends to opted-out numbers. Don't try to bypass it.
3. **Tech photo URLs.** Field-service CRMs often store tech photos as authenticated URLs that aren't public. Either move them to a public bucket (S3 + signed URL with long expiry, or Cloudinary) or skip the photo.
4. **Timing.** Send the SMS at dispatch, not at "tech accepted the job" or "tech started driving" — the customer cares about the window-to-arrival, not the tech's workflow state.
5. **Reply handling.** Customers will text back. Either route replies to your dispatcher's phone, or to a shared inbox like AgentMail. Don't let them go to a black hole.

## Variations

- **With live ETA updates.** Use the tech's phone GPS via the CRM (most field-service CRMs expose this) to send a second SMS "arriving in 5 min" when they're within a 1-mile geofence.
- **Without a CRM.** If you dispatch manually, build a tiny dispatcher app (n8n form node + Twilio node, 10 minutes) where the dispatcher types in customer phone + tech name + ETA, hits submit, and the SMS fires.
- **Multilingual.** Detect customer language from CRM record, swap the template.

## Production version

If you want this fully built, integrated with your CRM (including consent capture, reply routing, photo handling, multi-language, and analytics), that's the kind of build [Avalux](https://avalux.io) ships in 2-3 weeks. Pricing starts at $5,000. See [avalux.io/contact.html](https://avalux.io/contact.html) or email [eli@avalux.io](mailto:eli@avalux.io).

## License

MIT — part of the [solved-by-avalux](https://github.com/elikem2021/solved-by-avalux) cookbook.
