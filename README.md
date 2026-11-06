# searchres-worker

Cloudflare Worker search resolver and search result proxy.

## Purpose

`searchres-worker` is a serverless search endpoint and edge search worker built for Cloudflare Workers. The worker accepts incoming HTTP queries, scrapes DuckDuckGo web and image results over standard HTTPS, and returns structured JSON responses.

Many search scrapers rely on headless browsers such as Puppeteer or Playwright. Those browsers consume excessive memory and CPU, and search engines block Cloudflare data center IP addresses with CAPTCHAs during browser execution. This worker uses lightweight HTTP scraping libraries instead. It needs no browser instances, runs within standard Cloudflare Workers CPU limits, and requires no external search engine API keys.

## Architecture

The service runs on Cloudflare Workers using the V8 workerd runtime with Node.js compatibility enabled.

- Runtime: Cloudflare Workers ES module format.
- Resolvers: `ddg-search` parses HTML text results with Cheerio. `@mudbill/duckduckgo-images-api` queries DuckDuckGo image endpoints.
- Network fetch: The worker binds native Cloudflare `fetch` into the resolver via `fetch.bind(globalThis)`. Requests travel directly across the Cloudflare global network without Node.js stream wrappers.
- Cache behavior: The worker performs a live search request on each incoming query. It does not store search responses in Cloudflare Cache API or Workers KV. Preflight CORS responses include an `Access-Control-Max-Age` header of 86400 seconds, instructing browsers to cache preflight decisions for 24 hours.
- Cross-origin support: The worker attaches CORS headers to all responses, allowing web applications on any origin to query the API.

## Setup

The Cloudflare Worker source code and configuration reside in the `google-search-worker` directory.

### Prerequisites

- Node.js 18 or later
- npm
- Cloudflare account

### Installation

1. Clone the repository and navigate to the worker directory:
   ```bash
   git clone https://github.com/devsanthoshmk/searchres-worker.git
   cd searchres-worker/google-search-worker
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Authenticate Wrangler with Cloudflare:
   ```bash
   npx wrangler login
   ```

### Local development

Start the local development server:
```bash
npx wrangler dev
```

Wrangler starts a local HTTP server at `http://localhost:8787`.

### Deployment

Deploy the worker to your Cloudflare Workers account:
```bash
npx wrangler deploy
```

Wrangler prints the live deployment URL upon completion (for example, `https://searchres.<your-subdomain>.workers.dev`).

## Usage

The worker provides two search modes: web search and image search.

### Web search

Web search handles queries sent to `/`, `/search`, or any path other than `/images`.

#### Query parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `q` | string | Yes | None | Search query string |
| `pages` | integer | No | `1` | Number of result pages to scrape |
| `max` | integer | No | All | Maximum total result items to return |
| `region` | string | No | `wt-wt` | Geographic region code (for example, `en-US` or `uk-en`) |
| `time` | string | No | None | Time filter: `d` (past day), `w` (past week), `m` (past month) |

#### Example request

```bash
curl -s "https://searchres.sanpro.workers.dev/?q=cloudflare+workers&max=2"
```

#### Example response

```json
{
  "query": "cloudflare workers",
  "features_exposed": [
    "results",
    "zeroClick",
    "spelling",
    "pagesScraped"
  ],
  "data": {
    "results": [
      {
        "title": "Cloudflare Workers",
        "url": "https://workers.cloudflare.com/",
        "description": "Cloudflare Workers provides a serverless execution environment that runs your code across global data centers.",
        "displayUrl": "workers.cloudflare.com"
      }
    ],
    "spelling": null,
    "zeroClick": null,
    "pagesScraped": 1,
    "query": "cloudflare workers"
  }
}
```

### Image search

Image search handles queries sent to `/images` or requests containing the query parameter `type=image`.

#### Query parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `q` | string | Yes | None | Image search query string |
| `type` | string | No | None | Set to `image` to route to image search on any pathname |
| `safe` | boolean | No | `true` | Safe search filter. Pass `safe=false` to disable |
| `pages` | integer | No | `1` | Pagination iterations (each iteration fetches up to 100 images) |

#### Example request

```bash
curl -s "https://searchres.sanpro.workers.dev/images?q=red+panda&safe=true"
```

#### Example response

```json
{
  "query": "red panda",
  "features_exposed": [
    "image",
    "thumbnail",
    "title",
    "source",
    "url"
  ],
  "data": {
    "results": [
      {
        "image": "https://example.com/red-panda.jpg",
        "thumbnail": "https://tse1.mm.bing.net/th/id/example",
        "title": "Red Panda in Tree",
        "url": "https://example.com/animals/red-panda",
        "width": 1920,
        "height": 1080,
        "source": "Bing"
      }
    ]
  }
}
```

### Cross-origin requests

Clients can send cross-origin requests from any domain. The worker handles `OPTIONS` preflight requests and returns these response headers:

- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET,HEAD,POST,OPTIONS`
- `Access-Control-Allow-Headers: Content-Type, Authorization`
- `Access-Control-Max-Age: 86400`

### Error responses

The worker returns JSON error payloads with standard HTTP status codes.

Missing query parameter:
- Status: `400 Bad Request`
- Trigger: Missing or empty `q` parameter.

```json
{
  "error": "Missing 'q' query parameter"
}
```

Upstream error or unhandled exception:
- Status: `500 Internal Server Error`
- Trigger: Network failure or upstream response parsing failure.

```json
{
  "error": "Error: upstream request failed",
  "stack": "Error: upstream request failed\n    at search ..."
}
```

## Configuration

The worker configuration file is `google-search-worker/wrangler.json`.

```json
{
  "name": "searchres",
  "main": "src/index.ts",
  "compatibility_date": "2024-10-01",
  "compatibility_flags": ["nodejs_compat"]
}
```

Configuration parameters:
- `name`: Cloudflare Worker application name.
- `main`: Path to the TypeScript entry file (`src/index.ts`).
- `compatibility_date`: Cloudflare runtime version date.
- `compatibility_flags`: Runtime compatibility flags. The `nodejs_compat` flag provides the Node.js runtime globals required by HTML parsers.

The application does not require environment variables, secrets, or storage bindings.
