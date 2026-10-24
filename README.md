# n8n workflows for hotel prices: Booking.com and Google Hotels to Google Sheets

> The workflows call paid Apify Actors built by Flatrow Data Studio (the author). Links to apify.com carry no affiliate code.

Two ready-to-import [n8n](https://n8n.io) workflows that fill a Google Sheet with hotel prices every morning, without writing or hosting a scraper:

| Workflow | File | One row per |
|---|---|---|
| Track Booking.com hotel prices daily in Google Sheets | [`booking-hotel-prices-to-google-sheets.json`](booking-hotel-prices-to-google-sheets.json) | property: total price of the stay with and without taxes, review score, reviews, stars, link |
| Track hotel rate parity from Google Hotels in Google Sheets | [`google-hotels-rate-parity-to-google-sheets.json`](google-hotels-rate-parity-to-google-sheets.json) | hotel: cheapest booking site and its price, price on the hotel's own website, gap between the two |

Both use native n8n nodes only, with no community node to install: **Schedule Trigger → HTTP Request → Edit Fields (Set) → Google Sheets**, plus sticky notes that explain each step. The HTTP Request node runs an Apify Actor, the [Booking.com Hotels Scraper & Price Tracker](https://apify.com/myth84/booking-hotels-scraper) or the [Google Hotels Scraper & Price Tracker](https://apify.com/myth84/google-hotels-scraper), through Apify's synchronous endpoint `run-sync-get-dataset-items`: Apify runs the scraper in its cloud, waits for the end of the run and returns its rows, which n8n turns into one item per hotel. No login, no cookies, and **no personal data**: nothing about guests or reviewers.

## Import

1. In n8n, create a workflow, open the **...** menu at the top right, choose **Import from URL** and paste the raw link of a workflow:
   - `https://raw.githubusercontent.com/flatrow-data-studio/n8n-workflows/main/booking-hotel-prices-to-google-sheets.json`
   - `https://raw.githubusercontent.com/flatrow-data-studio/n8n-workflows/main/google-hotels-rate-parity-to-google-sheets.json`

   You can also download the file and choose **Import from File**, or copy its content and paste it on the canvas.
2. Create the two credentials of the next section and select them in the two nodes that ask for one.
3. Create an empty Google Sheet and paste its URL in the **Document** field of **Append rows to Google Sheets**. The rows go to the first tab of the sheet.
4. Click **Execute workflow**. 20 results take about a minute; the first run writes the header row, then one row per hotel.
5. Activate (publish) the workflow: it runs every day at 7:00, in the time zone of your n8n instance.

The nodes use the versions current in October 2026 (Schedule Trigger 1.3, HTTP Request 4.4, Edit Fields 3.4, Google Sheets 4.7). If your n8n reports an unknown node version on import, update n8n.

## Credentials

| Credential | Node | How to create it |
|---|---|---|
| Header Auth (Apify) | **Run Booking.com Hotels Scraper** or **Run Google Hotels Scraper** | Name `Authorization`, value `Bearer <your-apify-token>`. Copy the token from Apify Console > Settings > API & Integrations; a free Apify account is enough. |
| Google Sheets | **Append rows to Google Sheets** | A Google Sheets credential: on n8n Cloud, sign in with Google; on a self-hosted n8n, follow n8n's [Google credentials guide](https://docs.n8n.io/integrations/builtin/credentials/google/). |

The token lives in the n8n credential, never in the workflow: the JSON files contain no API key, no credential id and no sheet id.

## Booking.com hotel prices to Google Sheets

Every morning, the workflow searches Booking.com for one destination and a 3-night stay that starts 30 days later, for 2 adults, prices in euros, and appends one row per property. After a few days, the sheet is a price history: filter on `title` and chart `priceTotalAllIn` by `date`. Each run prices a stay one month ahead of the day it runs (`checkIn`), so the rows of one day compare hotels for the same stay, and the rows of one hotel show how the price of a stay one month ahead moves.

The body that the HTTP Request node sends to the Actor:

```json
{
  "locations": ["Lisbon, Portugal"],
  "checkInDaysFromNow": 30,
  "nights": 3,
  "adults": 2,
  "currency": "EUR",
  "maxResults": 20
}
```

## Booking.com sheet columns

| Column | Meaning |
|---|---|
| `date` | Day of the run (UTC), from the Actor's `scrapedAt` |
| `checkIn` | Check-in date of the priced stay |
| `searchLocation` | Destination of `locations` that found the property |
| `title` | Property name |
| `priceTotal` | Total price of the stay, before the taxes and fees Booking.com shows as excluded |
| `priceTotalAllIn` | Total price of the stay, taxes and fees included |
| `currency` | Currency of both prices |
| `rating` | Review score out of 10 |
| `reviewCount` | Number of reviews |
| `starRating` | Official stars |
| `url` | Property page with the dates and guests of the search |

One real row, from a Lisbon search (EUR, 20–23 November 2026, 2 adults), as the **Keep the sheet columns** node passes it to Google Sheets:

```json
{
  "date": "2026-10-19",
  "checkIn": "2026-11-20",
  "searchLocation": "Lisbon, Portugal",
  "title": "Villa Baixa - Lisbon Luxury Apartments",
  "priceTotal": 637.43,
  "priceTotalAllIn": 699.68,
  "currency": "EUR",
  "rating": 9.6,
  "reviewCount": 874,
  "starRating": 4,
  "url": "https://www.booking.com/hotel/pt/villa-baixa-lisbon-luxury-apartments.html?checkin=2026-11-20&checkout=2026-11-23&group_adults=2&group_children=0&no_rooms=1&selected_currency=EUR&lang=en-us"
}
```

## Google Hotels rate parity to Google Sheets

Every morning, the workflow searches Google Hotels for "hotels in Lisbon" and a 2-night stay that starts 30 days later, with `includeOffers` set to `true`: for each hotel, the Actor also reads the price of every booking site, the hotel's own website included. The **Compute the rate parity gap** node keeps the cheapest booking site, the price on the hotel's website and the gap between the two, so that a hotelier or a revenue manager sees at a glance where an online travel agency sells below the official rate.

- `officialSiteGap` is 0 when the hotel's website is the cheapest, or tied with the cheapest site.
- Above 0, a booking site sells the same stay for less than the hotel's website.
- It is empty when Google shows no rate from the hotel's website.

The body that the HTTP Request node sends to the Actor:

```json
{
  "searchQueries": ["hotels in Lisbon"],
  "checkInDaysFromNow": 30,
  "nights": 2,
  "adults": 2,
  "currency": "USD",
  "maxResults": 20,
  "includeOffers": true
}
```

## Google Hotels sheet columns

| Column | Meaning |
|---|---|
| `date` | Day of the run (UTC), from the Actor's `scrapedAt` |
| `checkIn` | Check-in date of the priced stay |
| `title` | Hotel name |
| `starRating` | Hotel class in stars |
| `currency` | Currency of every price of the row |
| `cheapestOfferSite` | Booking site with the lowest price for the stay |
| `cheapestOfferPriceTotal` | Its price for the whole stay, taxes included |
| `officialSitePriceTotal` | Price of the stay on the hotel's own website, taxes included |
| `officialSiteGap` | `officialSitePriceTotal` minus `cheapestOfferPriceTotal` |
| `officialSiteGapPercent` | The same gap, in % of `cheapestOfferPriceTotal` |
| `offerCount` | Number of booking sites with a rate |
| `url` | Google Hotels page of the hotel for the searched stay |

One real row, from a Lisbon search (USD, 20–23 November 2026, 2 adults), as the **Compute the rate parity gap** node passes it to Google Sheets:

```json
{
  "date": "2026-10-22",
  "checkIn": "2026-11-20",
  "title": "Brown's | Avenue Hotel",
  "starRating": 5,
  "currency": "USD",
  "cheapestOfferSite": "Vio.com",
  "cheapestOfferPriceTotal": 564.25,
  "officialSitePriceTotal": 565.73,
  "officialSiteGap": 1.48,
  "officialSiteGapPercent": 0.3,
  "offerCount": 22,
  "url": "https://www.google.com/travel/hotels/entity/ChkIhLCQwvjO2IYWGg0vZy8xMXE0bTZieDkyEAE?q=hotels%20in%20Lisbon&ts=CAESCgoCCAMKAggDEAAaIAoCGgASGhIUCgcI6g8QCxgUEgcI6g8QCxgXGAMyAhAAKgkKBToDVVNEGgA&hl=en&gl=us"
}
```

## Change the city, the dates or the schedule

Edit the **JSON** body of the HTTP Request node:

- **Destinations**: `locations` for Booking.com (`"Paris, France"`, written as on the site), `searchQueries` for Google Hotels (`"hotels in Rome"`). Both take several entries; `maxResults` applies to each one.
- **Stay**: `checkInDaysFromNow` and `nights` count from the day of the run, so the workflow never expires. For one fixed stay, replace them with `checkIn` and `checkOut` dates (YYYY-MM-DD).
- **Guests and currency**: `adults`, `currency`.
- **Booking.com filters**: `starRatings`, `minReviewScore`, `propertyTypes`, `freeCancellation`, `sortBy`, among others.

Every input field is documented on the Actor pages: [Booking.com Hotels Scraper & Price Tracker](https://apify.com/myth84/booking-hotels-scraper), [Google Hotels Scraper & Price Tracker](https://apify.com/myth84/google-hotels-scraper). Change the hour or the frequency in the **Every day at 7:00** node.

## Cost

Both Actors are pay-per-event: you pay a fixed price per result, with compute and proxies included. The Apify Free plan includes $5 of usage every month.

| Workflow | Price | One daily run of 20 results |
|---|---|---|
| [Booking.com](https://apify.com/myth84/booking-hotels-scraper) | $2.50 per 1,000 properties on the Apify Free plan, from $1.69 on paid plans | about $0.05 per run, about $1.50 a month |
| [Google Hotels](https://apify.com/myth84/google-hotels-scraper) | $2.50 per 1,000 hotels on the Apify Free plan, from $1.69 on paid plans, booking-site prices included | about $0.05 per run, about $1.50 a month |

`maxResults` caps the cost of every run: you pay only for the results returned, up to `maxResults` per destination.

## FAQ

**Why an HTTP Request node rather than an Apify node?** Nothing to install, and it works the same on n8n Cloud and on a self-hosted n8n. The Actor id is written with a tilde in the URL (`myth84~booking-hotels-scraper`).

**Why a 300-second timeout?** The HTTP Request node gives up after 10 seconds by default, and Apify's synchronous endpoint waits up to 5 minutes for the end of the run. A run of 20 results takes about a minute. For hundreds of hotels, lower the frequency of runs or split the destinations across several workflows.

**What happens when a search finds nothing?** The Actors never end a run "successfully" with no row: a destination they cannot recognise, dates in the past or a blocked search end as a failed run with an explicit message, visible under Runs in the Apify Console, and nothing is appended to the sheet.

**Is there personal data in the sheet?** No. The rows contain business facts (hotel names, prices, average scores and review counts), never guests, reviewers or review texts. Respect the terms of the sites and the law of your country, and do not republish listings.

## Related

- [booking-scraper-python](https://github.com/flatrow-data-studio/booking-scraper-python) and [google-hotels-scraper-python](https://github.com/flatrow-data-studio/google-hotels-scraper-python): the same Actors from Python, with a CSV export and a price tracker.
- [mcp-kits](https://github.com/flatrow-data-studio/mcp-kits): the same data in Claude, ChatGPT, Cursor or VS Code through Apify's MCP server.
