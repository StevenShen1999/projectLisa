# Our Tango

API documentation for the Our Tango E2E encrypted messaging app.

## Documentation Files

| File | Covers | Spec |
|------|--------|------|
| `openapi.yaml` | REST API (Auth, Users, Keys, Chats, Media, Notifications) | OpenAPI 3.0.3 |
| `asyncapi.yaml` | WebSocket events (messaging, typing, presence, receipts) | AsyncAPI 3.0.0 |

## Viewing the Docs

### OpenAPI (REST)

**Option 1 — Swagger UI (Docker)**
```bash
docker run -p 8081:8080 -e SWAGGER_JSON=/spec/openapi.yaml -v $(pwd):/spec swaggerapi/swagger-ui
```
Then open http://localhost:8081.

**Option 2 — Redoc (npx)**
```bash
npx @redocly/cli preview-docs openapi.yaml
```
Opens a browser at http://localhost:8080 with a rendered doc page.

**Option 3 — VS Code**
Install the [Swagger Viewer](https://marketplace.visualstudio.com/items?itemName=Arjun.swagger-viewer) or [OpenAPI (Swagger) Editor](https://marketplace.visualstudio.com/items?itemName=42Crunch.vscode-openapi) extension, then open `openapi.yaml` and preview.

### AsyncAPI (WebSocket)

**Option 1 — AsyncAPI Studio (online)**
Go to https://studio.asyncapi.com and paste the contents of `asyncapi.yaml`.

**Option 2 — AsyncAPI CLI (npx)**
```bash
npx -y @asyncapi/cli start studio
```
Opens AsyncAPI Studio locally in your browser.

**Option 3 — Generate static HTML**
```bash
npx -y @asyncapi/cli generate fromTemplate asyncapi.yaml @asyncapi/html-template -o docs/asyncapi
```
Then open `docs/asyncapi/index.html`.

## Linting / Validation

```bash
# Validate OpenAPI spec
npx @redocly/cli lint openapi.yaml

# Validate AsyncAPI spec
npx -y @asyncapi/cli validate asyncapi.yaml
```

## Frontend Integration

### REST API

Use the OpenAPI spec to generate a typed client:

```bash
# TypeScript (e.g. React, Vue, Svelte)
npx openapi-typescript openapi.yaml -o src/api/schema.d.ts

# Or generate a full client with openapi-fetch
npm install openapi-fetch
npx openapi-typescript openapi.yaml -o src/api/schema.d.ts
```

Then in your frontend code:
```ts
import createClient from 'openapi-fetch';
import type { paths } from './api/schema';

const api = createClient<paths>({ baseUrl: 'http://localhost:8080' });

// Fully typed — autocomplete on paths, params, and responses
const { data, error } = await api.POST('/api/auth/login', {
  body: { username: 'alice', password: 'securepass' },
});
```

### WebSocket

Use the AsyncAPI spec to generate typed message handlers:

```bash
npx -y @asyncapi/cli generate fromTemplate asyncapi.yaml @asyncapi/typescript-template -o src/api/ws
```

Or manually connect using the envelope format:
```ts
const ws = new WebSocket(`wss://api.example.com/ws?token=${accessToken}`);

// Send
ws.send(JSON.stringify({
  event: 'message:send',
  payload: {
    recipient_id: '...',
    ciphertext: '...',
    message_type: 'Text',
    client_message_id: crypto.randomUUID(),
  },
  request_id: crypto.randomUUID(),
}));

// Receive
ws.onmessage = (e) => {
  const { event, payload } = JSON.parse(e.data);
  switch (event) {
    case 'message:new':     // handle incoming message
    case 'message:delivered': // handle delivery receipt
    case 'typing:update':   // handle typing indicator
    case 'presence:update': // handle online/offline
    // ...
  }
};
```
