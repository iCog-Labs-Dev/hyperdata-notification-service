# Hyperdata Notification Service

NestJS worker that consumes notification events from RabbitMQ, stores notification records in the shared PostgreSQL database, and can send push notifications through OneSignal.

## How it works

```text
Hyperdata backend
      │ publishes notification.created
      ▼
RabbitMQ topic exchange → durable notifications.queue
                                │
                                ▼
                    Notification service
                      ├─ validates event
                      ├─ stores notification in PostgreSQL
                      └─ schedules OneSignal push (optional)
```
## Architecture
```
┌─────────────────┐
│  Mahder Backend   │
└────────┬────────┘
         │ Publish Events
         ▼
┌─────────────────┐
│    RabbitMQ     │
└────────┬────────┘
         │ Consume
         ▼
┌─────────────────────────────────┐
│   Notification Microservice     │
│  ┌──────────────────────────┐   │
│  │  RabbitMQ Consumer       │   │
│  └──────────┬───────────────┘   │
│             ▼                   │
│  ┌──────────────────────────┐   │
│  │  Notification Service    │   │
│  └──────────┬───────────────┘   │
│             ▼                   │
│  ┌──────────────────────────┐   │
│  │     PostgreSQL DB        │   │
│  └──────────────────────────┘   │
│             │                   │
└─────────────────────────────────┘
              │              
              ▼              
       ┌─────────────┐
       │Mobile Client│
       └─────────────┘
```

The backend remains the application API used by the web and mobile clients. This service currently exposes a root greeting, health check, and Swagger UI; it does not currently register notification REST controllers or a Socket.IO gateway. Do not point client notification-list or read-state calls at this service.

## Requirements and local setup

- Node.js 20 or later (the Docker image uses Node 20; CI builds with Node 22)
- pnpm 9
- PostgreSQL and RabbitMQ reachable from the service
- A database schema initialized by the Hyperdata backend migrations

Install dependencies and create a local environment file:

```bash
pnpm install --frozen-lockfile
cp .env.example .env
```

In PowerShell, use `Copy-Item .env.example .env`. Edit `.env` before starting the service. `DB_URL` must point to the database and schema created by the backend migrations. When running in the shared Docker network, use reachable service names such as `postgres` and `rabbitmq`; `localhost` inside a container refers to that container.

```bash
pnpm run start:dev
```

The service listens on `PORT` (default `3002`). This repository does not include a standalone development Compose stack; start PostgreSQL and RabbitMQ separately or use an approved shared development environment.

## Configuration

`.env.example` documents the configuration names. The variables required by startup validation are:

| Variable | Purpose |
|---|---|
| `PORT` | HTTP port; defaults to `3002` |
| `NODE_ENV` | `development`, `production`, or `test` |
| `DB_URL` | PostgreSQL connection URL used by TypeORM |
| `DATABASE_SCHEMA` | Existing PostgreSQL schema used by TypeORM |
| `RABBITMQ_HOST`, `RABBITMQ_PORT` | RabbitMQ address |
| `RABBITMQ_USERNAME`, `RABBITMQ_PASSWORD` | Dedicated RabbitMQ service credentials |
| `RABBITMQ_QUEUE` | Durable queue consumed by this worker |
| `RABBITMQ_EXCHANGE` | Durable topic exchange to bind |
| `RABBITMQ_ROUTING_KEY` | Binding key; normally `notification.created` |

Optional settings:

| Variable | Purpose |
|---|---|
| `ONESIGNAL_APP_ID` | OneSignal app identifier; enables push when paired with the REST key |
| `ONESIGNAL_REST_API_KEY` | Server-side OneSignal credential; never include it in the mobile app or source control |
| `CORS_ORIGIN` | Explicit trusted browser origin for the HTTP app; set a concrete origin in production |
| `LOG_LEVEL` | `error`, `warn`, `info`, `debug`, or `verbose` |
| `JWT_SECRET` | Present in the environment template for shared authentication integrations; it is not needed by the current health/root HTTP routes |

`DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD`, and `DB_DATABASE` are also present in the template. TypeORM connects using `DB_URL`; keep the values consistent. Replace all template credentials with unique values for the environment. Use a least-privileged database role and RabbitMQ account, and store production values in the deployment secret store rather than `.env` files in source control.

## RabbitMQ event contract

The consumer subscribes to routing key `notification.created`. On startup, the service attempts to create or verify the durable topic exchange, durable queue, and binding.

```json
{
  "userId": "<backend-user-id>",
  "notificationType": "task-assign",
  "title": "Task assigned",
  "message": "A task is ready for you",
  "payload": { "taskId": "<task-id>" }
}
```

| Field | Requirement | Notes |
|---|---|---|
| `userId` | Required string | Backend user ID; also used as the OneSignal external user ID |
| `notificationType` | Required enum | `task-assign`, `task-invitation`, `task-rejected`, or `task-approved` |
| `title` | Required string | Stored and used as the push heading |
| `message` | DTO-optional string | Include a non-empty value: the current PostgreSQL entity marks this column non-null, and it is used as the push content |
| `payload` | Optional object | Currently validated but not persisted or forwarded to OneSignal |

The backend may include `displayName`; this service does not currently consume it. Keep event fields free of credentials, access tokens, and unnecessary personal information. The consumer currently logs event data, so sensitive values in a message or payload may appear in service logs.

## Processing and delivery behavior

- Messages are manually acknowledged after notification persistence succeeds.
- Invalid events are acknowledged and discarded; inspect validation logs and correct the publisher's event shape.
- Persistence/processing exceptions are negatively acknowledged with requeue. A dead-letter queue is not configured, so a poison message can be retried repeatedly.
- A successful database write is the durable notification record. OneSignal delivery is scheduled asynchronously; push failures are logged and do not roll back the record or cause RabbitMQ redelivery.
- If a message is redelivered after a failure around acknowledgement, the service does not currently use an event idempotency key to deduplicate it. Publishers and operators should account for possible duplicate records when retrying.

## OneSignal and the mobile app

Set both `ONESIGNAL_APP_ID` and `ONESIGNAL_REST_API_KEY` on the notification service to enable pushes. If either is absent, notification persistence continues and push delivery is skipped. The REST API key is server-only.

The Flutter app initializes OneSignal with the same App ID and associates a signed-in user using the backend ID (`OneSignal.login(userId)`). It should call `OneSignal.logout()` when the user signs out. The service targets that external ID and sends `title`, `message`, plus `notificationId` and `notificationType` in the push data. The optional event `payload` is not included. Push acceptance by OneSignal is not proof that a device received or displayed the notification; check the device subscription and OneSignal delivery diagnostics.

There is no HTTP endpoint in this service for publishing a test notification. Test through the backend's configured publisher or OneSignal's dashboard with a subscribed test device; do not invent a `/notifications/test/publish` route.

## HTTP endpoints and health

With the default port:

- `GET http://localhost:3002/api/` — root greeting
- `GET http://localhost:3002/api/health` — HTTP and PostgreSQL health check
- `http://localhost:3002/api/docs` — Swagger UI

The health check currently verifies PostgreSQL only. A healthy response does not prove RabbitMQ connectivity or OneSignal delivery. Check the logs and service dependencies separately when diagnosing delivery issues.

## Logging and troubleshooting

The custom logger supports `error`, `warn`, `info`, `debug`, and `verbose`. Production logs use JSON; consumer events include correlation IDs. Inspect logs without printing `.env` files or secret values:

```bash
curl -fsS http://localhost:3002/api/health
docker logs --tail=100 <notification-container>
```

- **Database connection fails:** verify `DB_URL`, `DATABASE_SCHEMA`, network access, and that backend migrations created the notification table. TypeORM schema synchronization is disabled.
- **Queue remains empty:** confirm backend and worker use the same exchange, queue, binding key, RabbitMQ virtual host/network, and credentials.
- **Events are discarded:** check validation messages; use a supported `notificationType` and required fields.
- **Events repeatedly reappear:** inspect persistence errors and RabbitMQ redelivery; there is no configured dead-letter queue.
- **No push arrives:** verify both OneSignal server settings, the app's push permission/subscription, and that `userId` matches the mobile user's external ID.

Do not log tokens or dump entire event payloads to shared terminals when troubleshooting.

## Build, lint, and CI/CD

Available package scripts:

```bash
pnpm run lint
pnpm run build
pnpm run start:prod
```

Build the container image with:

```bash
docker build -t hyperdata-notification-service:local .
```

The image still needs the environment values and network access listed above. Production runs it through the shared Compose deployment rather than publishing the notification port directly.

The GitHub Actions workflow runs for pull requests and pushes to `main`, `master`, and `dev`. Lint is advisory (`continue-on-error`); the build is required. A push to `main` deploys the notification service through the shared production deployment script after the build succeeds.

Production runs this service in the backend's shared Docker Compose project and private network. Port `3002` is exposed to that network, not published directly to the host. Production environment values are loaded from the deployment's private `notification.env`; deployments use the `notification` target of `hyperdata-backend/deploy/deploy.sh`.

Useful source references:

- RabbitMQ consumer: [`src/rabbitmq/notification.consumer.ts`](src/rabbitmq/notification.consumer.ts)
- Event DTO: [`src/notifications/dto/notification-event.dto.ts`](src/notifications/dto/notification-event.dto.ts)
- Persistence and asynchronous push: [`src/notifications/notifications.service.ts`](src/notifications/notifications.service.ts)
- OneSignal client: [`src/notifications/services/onesignal.service.ts`](src/notifications/services/onesignal.service.ts)
- Health check: [`src/health/health.controller.ts`](src/health/health.controller.ts)
- Production Compose: [`../hyperdata-backend/deploy/docker-compose.production.yml`](../hyperdata-backend/deploy/docker-compose.production.yml)
