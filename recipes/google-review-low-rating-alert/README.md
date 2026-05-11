[![Maintained by Avalux](https://img.shields.io/badge/Maintained%20by-Avalux.io-3b82f6?style=flat-square)](https://avalux.io)

# Recipe: Instant Slack Alert When a Google Review Drops Below 4 Stars

> The problem with online reviews is that a single 1-star landing on a Friday at 4pm can sit unanswered all weekend, dragging your average from 4.7 to 4.3 by Monday and tanking your map-pack ranking. The fix is a 60-second poll → Slack alert with the review text and a direct response link.

## The problem

For local service businesses, Google reviews are one of the highest-leverage ranking signals in the map pack. A 1-star review that gets a thoughtful response within 2 hours often gets edited up to 3-4 stars by the customer. A 1-star review that sits for 3 days almost never does.

Most businesses learn about new reviews from a weekly digest email or when their owner happens to check Google. By then the damage is done.

## Who has this problem

- Multi-location service businesses (HVAC, plumbing, dental, optical, fitness)
- Local SMBs where Google reviews drive 30%+ of inbound calls
- Franchise operators monitoring 5-50 locations
- Anyone in the SMB ICP, honestly — this is universal

## When this fix makes sense

- You have at least one Google Business Profile (verified)
- You can dedicate someone (owner, ops manager, customer service) to respond within 1-2 hours
- You can stomach $0 of infrastructure cost (this runs on a $5 VPS)

## When it doesn't make sense

- You have no plan to respond to reviews (alert with no response is worse than not knowing)
- Your business is not local / not on Google Business Profile

## The free DIY path

Two moving parts:

1. **Poller** — every 5 minutes, query Google Business Profile API for new reviews across your locations
2. **Filter + alert** — if a new review is under 4 stars, send a Slack message with the review text, location, customer name, and a direct response link

Below is the core poll + alert in Python (~70 lines). For polygon-level alerting (e.g., only alert if review is for location A but not location B) just extend the locations list.

```python
# google_review_alert.py
import os, json, time, requests
from datetime import datetime, timezone
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request

GBP_TOKEN = os.environ["GBP_ACCESS_TOKEN"]
GBP_REFRESH = os.environ["GBP_REFRESH_TOKEN"]
GBP_CLIENT_ID = os.environ["GBP_CLIENT_ID"]
GBP_CLIENT_SECRET = os.environ["GBP_CLIENT_SECRET"]
GBP_ACCOUNT_ID = os.environ["GBP_ACCOUNT_ID"]
SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK_URL"]
LOCATIONS = json.loads(os.environ["GBP_LOCATIONS"])  # [{"id":"...","name":"Mississauga"}]
SEEN_FILE = "/var/lib/gbp-alert/seen.json"

THRESHOLD = 4  # alert on reviews below this star count


def fresh_creds():
    creds = Credentials(
        token=GBP_TOKEN,
        refresh_token=GBP_REFRESH,
        client_id=GBP_CLIENT_ID,
        client_secret=GBP_CLIENT_SECRET,
        token_uri="https://oauth2.googleapis.com/token",
    )
    if not creds.valid:
        creds.refresh(Request())
    return creds


def get_reviews(loc_id, creds):
    url = (
        f"https://mybusiness.googleapis.com/v4/accounts/"
        f"{GBP_ACCOUNT_ID}/locations/{loc_id}/reviews"
    )
    r = requests.get(url, headers={"Authorization": f"Bearer {creds.token}"}, timeout=15)
    r.raise_for_status()
    return r.json().get("reviews", [])


STAR_MAP = {"ONE": 1, "TWO": 2, "THREE": 3, "FOUR": 4, "FIVE": 5}


def main():
    creds = fresh_creds()
    seen = set(json.load(open(SEEN_FILE))) if os.path.exists(SEEN_FILE) else set()

    for loc in LOCATIONS:
        reviews = get_reviews(loc["id"], creds)
        for rv in reviews:
            review_id = rv["reviewId"]
            if review_id in seen:
                continue
            rating = STAR_MAP.get(rv["starRating"], 0)
            if rating >= THRESHOLD:
                seen.add(review_id)
                continue

            text = rv.get("comment", "(no comment)")[:400]
            reviewer = rv.get("reviewer", {}).get("displayName", "Anonymous")
            review_url = f"https://search.google.com/local/reviews?placeid={loc['id']}"

            requests.post(SLACK_WEBHOOK, json={
                "text": f"⚠️ New {rating}-star review for {loc['name']}",
                "attachments": [{
                    "color": "danger" if rating <= 2 else "warning",
                    "fields": [
                        {"title": "Reviewer", "value": reviewer, "short": True},
                        {"title": "Rating", "value": "⭐" * rating, "short": True},
                        {"title": "Review", "value": text},
                    ],
                    "actions": [{"type": "button", "text": "Respond on Google", "url": review_url}],
                }],
            }, timeout=10)
            seen.add(review_id)

    json.dump(list(seen), open(SEEN_FILE, "w"))


if __name__ == "__main__":
    main()
```

Run every 5 minutes via cron:

```cron
*/5 * * * * /usr/bin/python3 /opt/gbp-alert/google_review_alert.py
```

## Common gotchas

1. **Google Business Profile API is gated.** You need to apply for access (mybusiness.googleapis.com/v4) and Google takes 5-15 business days to approve. Apply via [Google Business Profile API access form](https://support.google.com/business/contact/api_default).
2. **Token refresh.** OAuth tokens expire every hour. The `google-auth` library handles this if you store the refresh token correctly.
3. **Star rating enum.** Google returns ratings as strings (`"ONE"`, `"TWO"`, ...) not integers. Don't forget the mapping.
4. **Anonymous reviews.** Some reviewers show up as "A Google User" with no `reviewer.displayName`. Handle gracefully.
5. **Review edits.** Google fires an update when a reviewer edits a review. If you also want to alert on edits (recommended — sometimes a 1-star becomes a 5-star after you respond), track `updateTime` separately from `createTime`.
6. **Rate limits.** GBP API caps at ~1 QPS per location. With 50 locations polling every 5 minutes you're well under, but if you scale to 500+ locations, batch or stagger.

## Variations

- **Sentiment-based alerting.** Don't just alert on star count — pass the text through Claude/GPT and alert on negative sentiment even if the star count is 4 (mixed reviews are often the most actionable).
- **Multi-platform.** Same pattern for Yelp, Facebook, TripAdvisor reviews. Each API is different but the alert layer is shared.
- **Auto-draft response.** Claude reads the review, drafts a thoughtful response in your brand voice, sends to Slack with an "Approve" button. One click → posts to Google.
- **Daily roll-up.** Same poller, weekly summary on Friday — what changed in average rating, response time, sentiment trend.

## Production version

[Avalux](https://avalux.io) builds the full review-monitoring stack — multi-platform, sentiment analysis, AI-drafted responses with approval workflow, ranking change tracking, and integration with your CRM so the customer's full history is in context. Pricing starts at $5,000. See [avalux.io/contact.html](https://avalux.io/contact.html) or email [eli@avalux.io](mailto:eli@avalux.io).

## License

MIT — part of the [solved-by-avalux](https://github.com/elikem2021/solved-by-avalux) cookbook.
