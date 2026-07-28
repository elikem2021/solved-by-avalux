[![Maintained by Avalux](https://img.shields.io/badge/Maintained%20by-Avalux.io-3b82f6?style=flat-square)](https://avalux.io)

# Solved by Avalux — Open-Source SMB Pain Recipes

> A growing library of working solutions for the operational pain points we run into most often when building automation for SMBs. Each recipe is a real problem, a plain-English explanation, working code, and the common gotchas we've hit. MIT-licensed. Use it, fork it, ship it.

This repo exists because the same problems keep showing up in small and mid-size businesses and most "solutions" online are either vendor pitches or theoretical write-ups. Here you get the actual code we use with clients, published so any in-house dev, ops lead, or founder can self-serve.

## Who maintains this

[Avalux](https://avalux.io) — a full-stack AI automation agency based in Mississauga, Ontario, working with SMBs in freight, e-commerce, home services, and adjacent verticals. We publish recipes from real client work (anonymized) and from our own ongoing R&D.

If a recipe almost-but-not-quite solves your problem, you have two options: fork it and adapt it yourself, or [hire us](https://avalux.io/contact.html) to build the production version. Both are valid.

## The recipes

### Home & field services

- [**tech-on-the-way-text**](./recipes/tech-on-the-way-text/) — Auto-text your customer with the tech's name, photo, and live ETA when a job is dispatched. Companion to most field-service CRMs (ServiceTitan, Jobber, Housecall Pro) via webhook + Twilio.

### Freight & logistics

- [**freight-eta-customer-text**](./recipes/freight-eta-customer-text/) — Geofence pickup/drop, auto-text receiver when truck is one hour out. Works on top of Geotab, Samsara, or Motive APIs. Companion to our [freight-eta-toolkit](https://github.com/avalux-io/freight-eta-toolkit).

### E-commerce

- [**shopify-qbo-line-item-sync**](./recipes/shopify-qbo-line-item-sync/) — Idempotent line-item sync from Shopify orders to QuickBooks Online with proper account mapping. Companion to our [shopify-quickbooks-sync](https://github.com/avalux-io/shopify-quickbooks-sync) toolkit.

### Reputation & reviews

- [**google-review-low-rating-alert**](./recipes/google-review-low-rating-alert/) — Get a Slack ping the moment a new Google review under 4 stars lands, with the review text and a link to respond. Works across multiple locations.

### Lead routing

- [**lead-router-by-postal-code**](./recipes/lead-router-by-postal-code/) — Inbound webhook → postal code lookup → route to the right rep's Slack/email with full context. Eliminates manual triage.

## How to use these

Each recipe directory has:
- `README.md` — the problem in plain English, who has it, when you should and shouldn't use this approach, common gotchas
- Working code (`script.py`, `workflow.json`, or `implementation.md`) you can copy directly
- A "where to put this in production" note

Most recipes are designed to be deployed on:
- A small VPS (Hetzner, Vultr, DigitalOcean — $5-10/month)
- A self-hosted [n8n](https://github.com/avalux-io/n8n-self-hosted-toolkit) instance
- Or wired directly into your existing stack via webhooks

## What this is not

- **Not a product.** No subscription, no signup, no telemetry. Clone it, fork it, do whatever.
- **Not generic AI slop.** Every recipe is a real pattern we've shipped in production. No "AI receptionist for HVAC" boilerplate.
- **Not closed source.** MIT-licensed. We make money by building the production version for clients who don't want to self-host.

## Contributing

Run into a problem one of these recipes solves but find a bug or want to add a variation? Open an issue or a PR. We review them.

## Related projects we maintain

- [freight-eta-toolkit](https://github.com/avalux-io/freight-eta-toolkit) — Geotab / Samsara / Motive utilities for SMB freight brokers
- [shopify-quickbooks-sync](https://github.com/avalux-io/shopify-quickbooks-sync) — Shopify ↔ QBO middleware
- [n8n-self-hosted-toolkit](https://github.com/avalux-io/n8n-self-hosted-toolkit) — Production n8n setup with hardened defaults
- [avalux-open-source](https://github.com/avalux-io/avalux-open-source) — Hub for everything we publish

## License

MIT. Use it, ship it, sell it. Attribution appreciated but not required.

## Working with Avalux

If you'd rather have any of these built, deployed, and operated for you — that's our day job. Fixed-scope contracts, real APIs only, monthly retainers for ongoing operations. Pricing starts at $5,000. See [avalux.io](https://avalux.io) or email [eli@avalux.io](mailto:eli@avalux.io).
