<p align="center"><a href="https://apify.com/DevilScrapes/luma-event-discovery"><img src="assets/banner.svg" alt="Lu.ma Event Discovery Scraper — Python example" width="100%"></a></p>

# Luma Event Scraper in Python: lu.ma city and category feeds into pandas

<p align="center">
<a href="https://apify.com/DevilScrapes/luma-event-discovery"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/luma-event-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/luma-event-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Many AI, crypto and startup communities announce their demo nights, founder dinners and hackathons on Luma (lu.ma), so its public city and category feeds work as a rough census of who is gathering where. This repository is a compact Python client for the **[Lu.ma Event Discovery Scraper](https://apify.com/DevilScrapes/luma-event-discovery)** on Apify. Point it at a slug such as `sf` or `ai`, or at a single event link, and it hands you a table where each row is one event with its start time and timezone, its city and region, its ticket pricing in cents, how many people registered and who is hosting. Apify runs the collection in its cloud; your script only waits for the run and writes `results.json` and `results.csv` for pandas, DuckDB or a spreadsheet.

> **Actor:** [Lu.ma Event Discovery Scraper](https://apify.com/DevilScrapes/luma-event-discovery) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [luma-event-scraper-javascript](https://github.com/devilscrapes/luma-event-scraper-javascript)

## 🔥 What you get

- **Feeds and single events in the same list**: a city slug like `sf`, a category like `ai`, the `discover` page or one event link can sit side by side in `urls`; the Actor tells list pages from event pages on its own.
- **Any spelling of the address works**: `lu.ma/...`, `luma.com/...` and bare slugs all resolve to the canonical `https://luma.com/<path>` form.
- **Money as integers**: `price_cents` and `max_price_cents` stay in minor units next to a `currency` code, so totals never drift through float rounding.
- **Turnout signals on every row**: `guest_count`, `ticket_count`, `is_sold_out` and `requires_approval` show how busy and how gated an event is.
- **Host context without extra calls**: the first host's display name, Luma username, verified flag and a LinkedIn link (only when it is a `/in/` profile), plus `host_count` for co-hosted events.
- **Speaker data only when asked**: `featured_guest_count` is always filled; the `featured_guests` list appears only with `includeFeaturedGuests` switched on.
- **Failures stay local**: each page gets up to five attempts with growing pauses on throttling and server errors, and a URL that still fails is logged and skipped while the others finish.

## 🎯 Use cases

### Weekly census of a city's tech calendar

Point cron at a handful of city slugs every Monday, append each CSV to a dated folder and chart events per week with `start_at`. After a month you can say with numbers whether a scene is growing, which weekdays it prefers and how much of it is paid.

### Rank events for a sponsorship budget

Load a category feed into a notebook, keep rows where `location_type` is `offline`, and sort by `guest_count`. Combine that with `is_sold_out` and `requires_approval` to separate the open crowd events from the curated rooms that a sponsor actually wants to be in.

### Price benchmarking for organizers

If you run events yourself, divide `max_price_cents` by 100 per city and category to see what comparable gatherings charge. The median and spread of paid tickets in your area take one `groupby` to compute, which beats guessing a price for your next workshop.

### Event-led prospect lists for research

Revenue and partnerships teams can filter a category for events hosted by verified profiles or co-hosted by several organizers, then review the `event_url` of each before reaching out. Keep the review human: the data says who organizes, not who wants to be contacted.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/luma-event-scraper-python.git
cd luma-event-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Lu.ma Event Discovery Scraper](https://apify.com/DevilScrapes/luma-event-discovery), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Here is the integration with nothing extra around it. The `apify-client` package launches the Actor, blocks until the run completes and then yields dataset rows one at a time.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/luma-event-discovery").call(run_input={'urls': ['sf'], 'maxEventsPerUrl': 10, 'includeFeaturedGuests': False, 'proxyConfiguration': {'useApifyProxy': True}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` submits `input.json` exactly as written. The bundled file reads the San Francisco feed and stops at ten events, so a test run costs cents. Swap in other slugs such as `nyc` or `ai`, or paste full event links; setting `maxEventsPerUrl` to `0` keeps every event the feed page holds.

```json
{
  "urls": [
    "sf"
  ],
  "maxEventsPerUrl": 10,
  "includeFeaturedGuests": false,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `urls` | array | yes | `["sf"]` | — |
| `maxEventsPerUrl` | integer | no | `100` | — |
| `includeFeaturedGuests` | boolean | no | `false` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Each row is a single Luma event. Prices are integers in minor units with a lowercase currency code beside them, and `address` comes back `null` whenever the organizer has hidden the venue from non-guests, so treat missing addresses as a privacy setting rather than a data gap.

| Field | Type | Always present |
|---|---|---|
| `source_url` | string | yes |
| `event_api_id` | string | yes |
| `name` | string | yes |
| `event_url` | string | yes |
| `start_at` | string | yes |
| `end_at` | string or null | no |
| `timezone` | string or null | no |
| `event_type` | string or null | no |
| `location_type` | string or null | no |
| `city` | string or null | no |
| `region` | string or null | no |
| `address` | string or null | no |
| `is_free` | boolean | no |
| `price_cents` | integer or null | no |
| `max_price_cents` | integer or null | no |
| `currency` | string or null | no |
| `is_sold_out` | boolean | no |
| `requires_approval` | boolean | no |
| `guest_count` | integer | no |
| `ticket_count` | integer | no |
| `host_name` | string or null | no |
| `host_username` | string or null | no |
| `host_linkedin` | string or null | no |
| `host_is_verified` | boolean | no |
| `host_count` | integer | no |
| `featured_guest_count` | integer | no |
| `featured_guests` | array or null | no |
| `cover_url` | string or null | no |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Result emitted | $0.004 |

Prices as configured on 2026-11-03. The [Actor page](https://apify.com/DevilScrapes/luma-event-discovery) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

This pandas snippet (`pip install pandas`) removes events that showed up under two feeds, converts prices to whole currency units, and prints a per-city summary followed by the ten biggest upcoming in-person events.

```python
import json

import pandas as pd

events = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
events = events.drop_duplicates(subset="event_api_id")
events["start"] = pd.to_datetime(events["start_at"], utc=True, errors="coerce")
events["top_price"] = events["max_price_cents"].astype("Float64") / 100

summary = events.groupby(events["city"].fillna("online / unknown")).agg(
    events=("event_api_id", "count"),
    paid_share=("is_free", lambda s: round(1 - s.mean(), 2)),
    median_guests=("guest_count", "median"),
    sold_out=("is_sold_out", "sum"),
)
print(summary.sort_values("events", ascending=False).to_string())

upcoming = events[(events["start"] >= pd.Timestamp.now(tz="UTC")) & (events["location_type"] == "offline")]
biggest = upcoming.sort_values("guest_count", ascending=False).head(10)
print(biggest[["start_at", "name", "city", "guest_count", "top_price", "event_url"]].to_string(index=False))
```

## ❓ FAQ

### Do I need a Luma account or a Luma API key?

No. This Actor reads pages anyone can open in a browser without signing in. An Apify API token is the only secret this script uses.

### How many events does one city slug return?

Roughly what Luma puts on that feed page on first load; the Actor does not click through further pages. Cover more ground by adding more slugs, such as neighbouring cities and topic categories, and dedupe on `event_api_id` afterwards.

### Can I see who registered for an event?

No, and that is by design. Attendee lists are not public on Luma, so the rows carry only totals like `guest_count`. Hosts appear because Luma shows them on the public page, and named speakers stay out of the output unless you turn on `includeFeaturedGuests`.

### Why does one event appear twice in my results?

An event can sit on a city feed and a category feed at once, and each input URL is processed independently, so both rows are written and billed. `source_url` tells you which feed produced each copy, and the recipe above drops the repeats.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/luma-event-discovery) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Lu.ma Event Discovery Scraper on Apify Store](https://apify.com/DevilScrapes/luma-event-discovery)
- [JavaScript version of this example](https://github.com/devilscrapes/luma-event-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Meetup Events Scraper](https://apify.com/DevilScrapes/meetup-events-scraper) — the same kind of rows from Meetup keyword searches, with RSVP totals and host groups.
- [Eventbrite Events Scraper](https://apify.com/DevilScrapes/eventbrite-events-scraper) — ticketed Eventbrite listings when your market lives there instead.
- [Meetup Group Scraper](https://apify.com/DevilScrapes/meetup-group-scraper) — a full upcoming and past event history for chosen Meetup groups.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/luma-event-discovery) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Luma (lu.ma).
