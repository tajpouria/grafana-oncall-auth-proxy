# Grafana OnCall Nginx Auth Proxy

Nginx reverse proxy with custom authentication and authorization for Grafana OnCall plugin operations.

## Overview

This proxy sits between clients and Grafana to enforce authentication and team-based authorization for OnCall shift swap operations while allowing unrestricted access to other Grafana features.

## Features

- **Session-based auth**: Validates Grafana session cookies via `/api/user` endpoint
- **Service account auth**: Supports Grafana service account tokens (prefix: `glsa_`)
- **Team-based authorization**: Restricts shift swap creation to users in the schedule's team
- **Admin bypass**: Grafana admins have unrestricted access
- **WebSocket support**: Proxies Grafana Live WebSocket connections

## Authentication Flow

1. **User requests**: Session cookie validated against Grafana API
2. **Service accounts**: Bearer tokens validated for `/api/plugins/grafana-oncall-app/resources/schedules` only
3. **Shift swaps**: Additional team membership check against schedule ownership
4. **Other requests**: Admin-only for POST operations to OnCall API

## Authorization Rules

| Request Type              | Authentication | Authorization           |
| ------------------------- | -------------- | ----------------------- |
| GET/OPTIONS to OnCall API | None           | Public                  |
| POST to shift swaps       | Session/Token  | Team member of schedule |
| Other POST to OnCall API  | Session        | Admin only              |
| Service account requests  | Token          | Schedules endpoint only |

## Configuration

### Required Dependencies

- Nginx with Lua support (`lua-nginx-module`)
- `lua-cjson` library
- Docker network with `grafana` container

### Environment

- Grafana container accessible at `grafana:3000`
- Docker DNS resolver at `127.0.0.11`

## Usage

### Docker Compose

```bash
docker compose up -d
docker compose logs nginx
```

### Standalone

```bash
nginx -c /path/to/nginx.conf -g "daemon off;"
```

## API Endpoints

- `GET /api/live/ws` - WebSocket proxy to Grafana Live
- `POST /api/plugins/grafana-oncall-app/resources/shift_swaps` - Team-restricted
- `GET /api/plugins/grafana-oncall-app/resources/*` - Public
- `/*` - Standard Grafana proxy

## Logging

Auth decisions logged to Nginx error log:

- User authentication status
- Team membership checks
- Authorization decisions
- Service account validation

## Development

Test authentication:

```bash
curl -H "Cookie: grafana_session=..." \
     -X POST http://localhost:3000/api/plugins/grafana-oncall-app/resources/shift_swaps \
     -d '{"schedule":"123"}'
```

Test service account:

```bash
curl -H "Authorization: glsa_token_here" \
     http://localhost:3000/api/plugins/grafana-oncall-app/resources/schedules
```
