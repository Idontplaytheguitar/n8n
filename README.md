# n8n Google Maps 5-Star Reviews Scraper

A self-hosted n8n workflow that scrapes Google Maps reviews using Apify, filters for 5-star positive reviews only, and exports them to JSON. Includes Cloudflare Tunnel for free public access.

## Prerequisites

- Docker and Docker Compose installed
- Free Apify account ([sign up here](https://console.apify.com/sign-up))
- Free Cloudflare account with a domain
- Google Maps place URL(s) to scrape

## Quick Start

### 1. Configure Environment Variables

Copy the example environment file and update with your values:

```bash
cp .env.example .env
```

Edit `.env` and set:
- `N8N_BASIC_AUTH_USER` - Username for n8n login
- `N8N_BASIC_AUTH_PASSWORD` - Password for n8n login (change from default!)
- `N8N_ENCRYPTION_KEY` - A 32-character string for encrypting credentials
- `APIFY_API_TOKEN` - Your Apify API token from [Apify Console](https://console.apify.com/account/integrations)
- `CLOUDFLARE_TUNNEL_TOKEN` - Your Cloudflare Tunnel token (see below)

### 2. Set Up Cloudflare Tunnel (Free Public Access)

This allows external services to call your n8n webhooks without opening ports.

1. Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. Navigate to **Networks > Tunnels**
3. Click **Create a tunnel**
4. Choose **Cloudflared** connector
5. Name it: `n8n-tunnel`
6. Copy the **tunnel token** (starts with `eyJ...`) and add to `.env`:
   ```
   CLOUDFLARE_TUNNEL_TOKEN=eyJ...your-token-here
   ```
7. In the tunnel setup, add a **Public Hostname**:
   - **Subdomain**: `n8n`
   - **Domain**: your domain
   - **Service Type**: `HTTP`
   - **URL**: `n8n:5678`

### 3. Start n8n

```bash
docker-compose up -d
```

n8n will be available at:
- **Local**: http://localhost:5678
- **Public**: https://n8n.yourdomain.com (via Cloudflare Tunnel)

### 4. Import the Workflow

1. Open n8n at http://localhost:5678
2. Log in with your credentials
3. Click on **Workflows** in the sidebar
4. Click **Import from File**
5. Select `workflows/google-maps-scraper.json`

### 5. Configure the Workflow

1. Open the imported workflow
2. Click on the **Configuration** node
3. Update the `googleMapsUrl` with your target Google Maps place URL
   - Example: `https://www.google.com/maps/place/Central+Park/@40.7828647,-73.9675438`
4. Adjust `maxReviews` if needed (default: 100)
5. **Activate the workflow** using the toggle in the top-right (required for webhook)
6. Save the workflow

### 6. Run the Workflow

**Manual execution:**
1. Click **Execute Workflow** to run manually
2. The exported JSON will be saved to the `output/` directory

**Via Webhook (for external integrations):**
```bash
curl -X POST https://n8n.yourdomain.com/webhook/reviews
```
Returns JSON with all 5-star reviews.

## Workflow Overview

The workflow supports two trigger modes:

```
Manual Trigger ──┐
                 ├─> Configuration -> Apify -> Extract -> Filter 5-Star -> Route
Webhook Trigger ─┘                                                           │
                                                                             ├─> (webhook) Respond to Webhook
                                                                             └─> (manual) Save to File
```

### Nodes Explained

| Node | Description |
|------|-------------|
| Manual Trigger | Starts the workflow manually |
| Webhook Trigger | Accepts POST requests at `/webhook/reviews` |
| Configuration | Set your Google Maps URL and max reviews |
| Fetch Reviews from Apify | Calls Apify's Google Maps Scraper API |
| Extract Reviews | Parses the response and extracts review data |
| Filter 5-Star Only | Keeps only reviews with exactly 5 stars |
| Prepare Response | Formats reviews for JSON output |
| Route Output | Routes to webhook response or file save |
| Respond to Webhook | Returns JSON response to HTTP caller |
| Save to JSON File | Writes the file to `/output` directory (manual only) |

## Output Format

The exported JSON has this structure:

```json
{
  "exportedAt": "2026-01-20T12:00:00.000Z",
  "totalReviews": 42,
  "filterCriteria": "5-star reviews only",
  "reviews": [
    {
      "placeName": "Central Park",
      "placeAddress": "New York, NY 10022",
      "placeUrl": "https://www.google.com/maps/place/...",
      "reviewerName": "John Doe",
      "reviewerId": "123456789",
      "reviewerUrl": "https://www.google.com/maps/contrib/...",
      "stars": 5,
      "reviewText": "Amazing place! Loved every moment...",
      "publishedAtDate": "2026-01-15",
      "reviewId": "abc123",
      "reviewUrl": "https://www.google.com/maps/reviews/...",
      "ownerResponse": null
    }
  ]
}
```

## Scheduling (Optional)

To run the scraper on a schedule:

1. Open the workflow in n8n
2. Add a **Schedule Trigger** node alongside the Manual Trigger
3. Configure your preferred schedule (daily, weekly, etc.)
4. Connect it to the Configuration node
5. Activate the workflow using the toggle in the top-right

## Apify Pricing

Apify offers a free tier with $5 USD monthly credits. The Google Maps Scraper typically costs:
- ~$0.50 per 1,000 reviews scraped
- Free tier is sufficient for moderate usage

Check current pricing at [Apify Pricing](https://apify.com/pricing).

## Troubleshooting

### "Invalid API token" error
Verify your `APIFY_API_TOKEN` in the `.env` file is correct.

### No reviews returned
- Check if the Google Maps URL is valid and accessible
- Ensure the place has reviews
- Some places may have restrictions on review scraping

### File not appearing in output folder
- Verify the Docker volume is mounted correctly
- Check n8n logs: `docker-compose logs n8n`

### Container won't start
```bash
docker-compose logs n8n
```

## Commands Reference

```bash
# Start n8n
docker-compose up -d

# Stop n8n
docker-compose down

# View logs
docker-compose logs -f n8n

# Restart n8n
docker-compose restart n8n

# Update n8n to latest version
docker-compose pull && docker-compose up -d
```

## File Structure

```
.
├── docker-compose.yml          # Docker configuration
├── .env                        # Environment variables (your config)
├── .env.example                # Example environment file
├── workflows/
│   └── google-maps-scraper.json   # n8n workflow to import
├── output/                     # Exported JSON files appear here
└── README.md                   # This file
```

## Security Notes

- Never commit `.env` to version control
- Use strong passwords for n8n authentication
- Keep your Apify API token secret
- Consider using a reverse proxy (nginx/traefik) with HTTPS for production
