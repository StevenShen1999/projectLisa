# Our Tango

API documentation for the Our Tango messaging app.

## Documentation Files

| File | Covers | Spec |
|------|--------|------|
| `openapi.yaml` | REST API (Auth, Contacts, Message History) | OpenAPI 3.0.3 |
| `asyncapi.yaml` | WebSocket events (send/receive messages) | AsyncAPI 3.0.0 |

## API Overview

### REST Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/auth/register` | — | Register a new user |
| `POST` | `/api/auth/login` | — | Login, returns JWT |
| `GET` | `/api/users/me/contacts` | JWT | Paginated contacts + latest message |
| `GET` | `/api/chats/{userId}/messages` | JWT | Paginated message history with a user |

### WebSocket

Connect at `ws(s)://host/ws?token=<jwt>`.

| Direction | Event | Description |
|-----------|-------|-------------|
| Client → Server | `message:send` | Send a message to a user |
| Server → Client | `message:new` | Receive an incoming message |

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

**Option 3 — VS Code**
Install the [Swagger Viewer](https://marketplace.visualstudio.com/items?itemName=Arjun.swagger-viewer) or [OpenAPI (Swagger) Editor](https://marketplace.visualstudio.com/items?itemName=42Crunch.vscode-openapi) extension, then open `openapi.yaml` and preview.

### AsyncAPI (WebSocket)

**Option 1 — AsyncAPI Studio (online)**
Go to https://studio.asyncapi.com and paste the contents of `asyncapi.yaml`.

**Option 2 — AsyncAPI CLI (npx)**
```bash
npx -y @asyncapi/cli start studio
```

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
npm install openapi-fetch
```

Then in your frontend code:
```ts
import createClient from 'openapi-fetch';
import type { paths } from './api/schema';

const api = createClient<paths>({ baseUrl: 'http://localhost:8080' });

// Login
const { data } = await api.POST('/api/auth/login', {
  body: { username: 'alice', password: 'securepass' },
});
const token = data?.access_token;
```

### WebSocket

```ts
const ws = new WebSocket(`wss://api.example.com/ws?token=${accessToken}`);

// Send a message
ws.send(JSON.stringify({
  event: 'message:send',
  payload: {
    recipient_id: '00000000-0000-0000-0000-000000000001',
    content: 'Hey!',
    client_message_id: crypto.randomUUID(),
  },
}));

// Receive messages
ws.onmessage = (e) => {
  const { event, payload } = JSON.parse(e.data);
  if (event === 'message:new') {
    console.log(`${payload.sender_id}: ${payload.content}`);
  }
};
```
