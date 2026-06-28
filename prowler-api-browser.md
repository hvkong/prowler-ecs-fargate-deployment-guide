# API Browser Access Guide

The Prowler API can be accessed directly from a browser's developer console or any HTTP
client. This is useful for automation, scripting, bulk operations, or debugging API
responses without going through the UI.

## When to Use

- Automating provider creation or scan triggers via scripts
- Extracting findings data for custom reporting pipelines
- Debugging UI issues by testing API endpoints directly
- Bulk operations not supported by the UI (e.g., listing all tenants)
- Integrating Prowler with external tools (SIEM, ticketing systems, dashboards)

## API Documentation

The full interactive API docs (Swagger/OpenAPI) are available at:
```
https://<your-domain>/api/v1/docs
```

## Authentication

The API supports two authentication methods:

### Option A: API Key (recommended for scripts)

Generate an API key in the Prowler UI: **Settings → API Keys → Create**.

```javascript
const PROWLER_API_KEY = 'YOUR_API_KEY_HERE';
const PROWLER_URL = 'your-domain.com';

fetch(`https://${PROWLER_URL}/api/v1/tenants`, {
  method: 'GET',
  headers: {
    'Authorization': `Api-Key ${PROWLER_API_KEY}`,
    'Content-Type': 'application/vnd.api+json'
  }
})
.then(response => {
  if (!response.ok) {
    throw new Error(`HTTP error! Status: ${response.status}`);
  }
  return response.json();
})
.then(data => console.log('Success:', data))
.catch(error => console.error('Error:', error));
```

### Option B: Bearer Token (JWT)

Obtain a token by authenticating with email and password:

```javascript
const PROWLER_URL = 'your-domain.com';

// Step 1: Get token
const tokenResponse = await fetch(`https://${PROWLER_URL}/api/v1/tokens`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/vnd.api+json' },
  body: JSON.stringify({
    data: {
      type: 'tokens',
      attributes: {
        email: 'your-email@example.com',
        password: 'your-password'
      }
    }
  })
});
const tokenData = await tokenResponse.json();
const accessToken = tokenData.data.attributes.access;

// Step 2: Use token
const response = await fetch(`https://${PROWLER_URL}/api/v1/providers`, {
  headers: {
    'Authorization': `Bearer ${accessToken}`,
    'Content-Type': 'application/vnd.api+json'
  }
});
const providers = await response.json();
console.log(providers);
```

## Common API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/tokens` | POST | Authenticate and get JWT tokens |
| `/api/v1/tenants` | GET | List tenants |
| `/api/v1/providers` | GET | List configured providers |
| `/api/v1/providers` | POST | Add a new provider |
| `/api/v1/scans` | GET | List scans |
| `/api/v1/scans` | POST | Trigger a new scan |
| `/api/v1/findings` | GET | List findings (supports filtering) |
| `/api/v1/overviews/findings` | GET | Findings summary dashboard data |
| `/api/v1/compliance-overviews` | GET | Compliance framework results |
| `/api/v1/resources` | GET | Discovered cloud resources |

## Filtering and Pagination

The API uses JSON:API specification. Filter and paginate like this:

```javascript
// Get page 2 with 50 items per page
fetch(`https://${PROWLER_URL}/api/v1/findings?page[number]=2&page[size]=50`, { ... })

// Filter by severity
fetch(`https://${PROWLER_URL}/api/v1/findings?filter[severity]=critical`, { ... })

// Filter by provider
fetch(`https://${PROWLER_URL}/api/v1/findings?filter[provider]=${providerId}`, { ... })
```

## Using from the Browser Console

Open DevTools (F12) on the Prowler UI page. Your session cookie is already set, so you
can make authenticated requests using the session's JWT:

```javascript
// This works from the browser console while logged into Prowler UI
const response = await fetch('/api/v1/providers', {
  headers: { 'Accept': 'application/vnd.api+json' }
});
const data = await response.json();
console.table(data.data.map(p => ({
  id: p.id,
  provider: p.attributes.provider,
  alias: p.attributes.alias,
  connected: p.attributes.connected
})));
```

## Using with curl

```bash
# With API key
curl -s -H "Authorization: Api-Key YOUR_API_KEY" \
  -H "Content-Type: application/vnd.api+json" \
  https://your-domain.com/api/v1/providers | jq .

# Trigger a scan
curl -s -X POST \
  -H "Authorization: Api-Key YOUR_API_KEY" \
  -H "Content-Type: application/vnd.api+json" \
  -d '{"data":{"type":"scans","attributes":{"trigger":"manual"},"relationships":{"provider":{"data":{"type":"providers","id":"PROVIDER_UUID"}}}}}' \
  https://your-domain.com/api/v1/scans | jq .
```

## Notes

- The API uses [JSON:API](https://jsonapi.org/) format — responses include `data`,
  `meta`, and `links` sections.
- Rate limiting applies: token obtain is limited to 50 requests/minute by default
  (`DJANGO_THROTTLE_TOKEN_OBTAIN`).
- API keys do not expire but can be revoked from the UI.
- JWT access tokens expire based on `DJANGO_ACCESS_TOKEN_LIFETIME` (default 180 minutes
  in this deployment).
