# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Evolution API is a comprehensive WhatsApp API controller built on Express.js and TypeScript. It began as a WhatsApp controller based on the Baileys library but has evolved into a multi-platform messaging API supporting WhatsApp (via Baileys and official Cloud API), with upcoming support for Instagram and Messenger.

**Core Technology Stack:**
- **Runtime**: Node.js ≥20, npm ≥9
- **Language**: TypeScript with CommonJS modules
- **Framework**: Express.js with custom module architecture
- **Database**: PostgreSQL via Prisma ORM
- **WhatsApp**: Baileys library (v6.7.19) for WhatsApp Web API
- **Caching**: Redis or local cache via configurable CacheEngine
- **Real-time**: Socket.io for WebSocket events
- **Message Queues**: RabbitMQ, Amazon SQS, NATS support

## Development Commands

### Building and Running
```bash
# Development with hot reload
npm run dev:server

# Type checking only (no build)
npm run build

# Production build
npm run build  # Then start with: npm run start:prod

# Start from source (development)
npm start

# Production start (requires build first)
npm run start:prod
```

### Code Quality
```bash
# Lint and auto-fix
npm run lint

# Lint check only (no fixes)
npm run lint:check
```

### Database Operations
```bash
# Generate Prisma client from schema
npm run db:generate

# Push schema changes to database (no migrations)
npm run db:deploy

# Open Prisma Studio for database inspection
npm run db:studio

# Development database push (alias)
npm run db:migrate:dev
```

### Testing
```bash
# Run all tests with watch mode
npm test
```

**Note**: This project uses `prisma db push` instead of migrations for schema management. Always run `db:generate` after schema changes.

## Architecture

### Module System

The application follows a **centralized module pattern** where all controllers, services, and integrations are instantiated in `src/api/server.module.ts` and exported for use throughout the app. This creates a dependency injection-like pattern without a formal DI framework.

**Key architectural files:**
- `src/main.ts` - Application bootstrap, Express setup, middleware configuration
- `src/api/server.module.ts` - Central module that instantiates all services, controllers, and manages dependency wiring
- `src/api/routes/index.router.ts` - Main router that aggregates all sub-routers

### Directory Structure

```
src/
├── api/
│   ├── abstract/         # Base classes and interfaces
│   ├── controllers/      # Request handlers (instance, chat, group, message, etc.)
│   ├── dto/              # Data Transfer Objects for validation
│   ├── guards/           # Auth and instance guards, telemetry
│   ├── integrations/     # External service integrations
│   │   ├── channel/      # WhatsApp (Baileys), Meta, Evolution channels
│   │   ├── chatbot/      # Typebot, Chatwoot, Dify, OpenAI, Flowise, N8N, EvoAI
│   │   ├── event/        # RabbitMQ, SQS, WebSocket, Webhook, NATS, Pusher
│   │   └── storage/      # S3/Minio integrations
│   ├── provider/         # Session file providers
│   ├── repository/       # Prisma database access layer
│   ├── routes/           # Route definitions per resource
│   ├── services/         # Business logic (monitor, auth, cache, channel, proxy, template, settings)
│   └── types/            # TypeScript type definitions
├── cache/                # Cache engine abstraction (Redis/Local)
├── config/               # Environment config, logger, error handling, event emitter
├── exceptions/           # Custom exception classes
├── utils/                # Utilities (JID creation, i18n, Sentry, telemetry, proxy)
└── validate/             # Validation schemas
```

### Core Architectural Patterns

**1. WAMonitoringService (`src/api/services/monitor.service.ts`)**
- Central service managing all WhatsApp instance lifecycles
- Maintains `waInstances` object mapping instance names to WhatsApp clients
- Handles connection state, QR code generation, event emission
- Orchestrates message handling and integration triggers

**2. Instance Management Flow**
```
Client Request → InstanceController → WAMonitoringService
                                      ↓
                              Create/Load Instance
                                      ↓
                              Baileys Client Init
                                      ↓
                              Event Listeners Setup
                                      ↓
                              Integration Triggers (Chatbots, Webhooks, etc.)
```

**3. Integration Architecture**
All integrations follow a controller/service pattern:
- **Base Classes**: `base-chatbot.controller.ts` and `base-chatbot.service.ts` provide common functionality
- **Specific Implementations**: Each integration (Typebot, Chatwoot, Dify, etc.) extends base classes
- **Trigger System**: Integrations can be triggered by keywords, message types, or advanced conditions
- **Session Management**: Integrations maintain session state in database via Prisma

**4. Event System**
- Uses EventEmitter2 for internal event bus (`@config/event.config.ts`)
- External events routed through `EventManager` to RabbitMQ/SQS/WebSocket/Webhook
- All WhatsApp events (messages, presence, chats, groups) can be forwarded to configured event destinations

**5. Multi-Channel Architecture**
- **Baileys Channel** (`src/api/integrations/channel/whatsapp/`): Free WhatsApp Web API
- **Meta Channel** (`src/api/integrations/channel/meta/`): Official WhatsApp Cloud API
- **Evolution Channel** (`src/api/integrations/channel/evolution/`): Custom protocol
- Channel selection per instance, managed by ChannelController

### Path Aliases (tsconfig.json)

TypeScript path aliases are configured for cleaner imports:
```typescript
@api/*      → ./src/api/*
@cache/*    → ./src/cache/*
@config/*   → ./src/config/*
@exceptions → ./src/exceptions
@utils/*    → ./src/utils/*
@validate/* → ./src/validate/*
```

Always use these aliases instead of relative imports when crossing module boundaries.

### Database Schema (Prisma)

**Key Models:**
- `Instance` - WhatsApp instance metadata (name, status, connection state, owner)
- `Message` - All messages sent/received with media references
- `Contact` - Contact information per instance
- `Chat` - Conversation metadata
- `IntegrationSession` - Active chatbot/integration sessions
- `Webhook` - Webhook configurations per instance
- `Proxy` - Proxy settings for instances
- Various chatbot-specific models (Typebot, Chatwoot, Dify, OpenAI, etc.)

**Important**: Schema path is `prisma/schema.prisma`. All database operations go through `PrismaRepository` service.

### Guards and Middleware

**Authentication Guards** (`src/api/guards/`):
- `auth.guard.ts` - API key validation (global or per-instance)
- `instance.guard.ts` - Two guards:
  - `instanceExistsGuard` - Validates instance exists
  - `instanceLoggedGuard` - Validates instance is connected
- `telemetry.guard.ts` - Collects usage analytics (can be disabled)

Guards are applied at the router level, not as Express middleware.

### Configuration System

Environment configuration in `src/config/env.config.ts` uses a factory pattern:
- Reads from `.env` file (use `.env.example` as template)
- Provides typed configuration objects for all services
- Config access via `configService.get<ConfigType>('CONFIG_NAME')`
- **Critical configs**: Database connection, authentication, CORS, integrations

## Working with WhatsApp Instances

### Instance Lifecycle
1. **Create**: POST to instance endpoint with name and optional settings
2. **Connect**: Instance generates QR code or connects with saved session
3. **Ready**: Instance status becomes "open", can send/receive messages
4. **Disconnect**: Graceful shutdown or connection lost
5. **Cleanup**: Sessions can be auto-deleted based on `DEL_INSTANCE` config

### Message Flow
```
Incoming WhatsApp Event → Baileys Client → WAMonitoringService
                                           ↓
                                    Event Processing
                                           ↓
                         ┌─────────────────┼─────────────────┐
                         ↓                 ↓                 ↓
                   Database Save    Integration Trigger  Event Emission
                   (if enabled)     (Chatbots, etc.)     (Webhooks, etc.)
```

### Integration Triggers

Chatbot integrations support multiple trigger types:
- **TriggerType.all** - Trigger on every message
- **TriggerType.keyword** - Trigger on specific keywords
- **TriggerType.advanced** - Custom trigger conditions with operators
- **TriggerType.none** - Manual trigger only

Trigger matching uses `findBotByTrigger` utility with support for:
- Exact match, contains, startsWith, endsWith, regex operators
- Case sensitivity options
- Priority-based matching when multiple triggers match

## Adding New Features

### New Integration Checklist
1. Create directory under `src/api/integrations/[category]/[name]/`
2. Implement controller extending appropriate base class
3. Implement service extending appropriate base class
4. Create DTOs for request/response validation
5. Add database models to `prisma/schema.prisma` if needed
6. Create router in integration directory
7. Register controller/service in `src/api/server.module.ts`
8. Add router to appropriate parent router
9. Add environment config in `src/config/env.config.ts`

### New Controller Checklist
1. Create controller class in `src/api/controllers/`
2. Inject required services via constructor
3. Implement methods with DTO validation
4. Create router in `src/api/routes/`
5. Apply appropriate guards (auth, instance, etc.)
6. Register controller in `server.module.ts`
7. Add router to `index.router.ts`

## Important Behaviors

### Sentry Integration
- Sentry must be initialized FIRST in `src/main.ts` (see line 2 import)
- Import order matters: `@utils/instrumentSentry` before other modules
- Only enabled if `SENTRY_DSN` environment variable is set

### Session Storage
- Sessions stored in `store/` directory by default
- Can be configured to use external providers (S3, Minio)
- Session files include auth state, keys, and message cache

### Cache Strategy
- Two-tier caching: Redis (if available) or in-memory fallback
- Separate cache instances for: instance data, Baileys sessions, Chatwoot state
- Cache engine selection in `CacheEngine` based on environment config

### Error Handling
- All routes wrapped with `express-async-errors`
- Custom error middleware in `src/main.ts`
- Errors can trigger webhooks if `WEBHOOK.EVENTS.ERRORS` is enabled
- HTTP status codes defined in `HttpStatus` enum

### Telemetry
- Collects anonymous usage data (routes accessed, API version)
- No sensitive data transmitted
- Can be disabled but enabled by default
- Implemented in `telemetry.guard.ts`

## Common Development Tasks

### Adding a new chatbot integration
Follow the pattern in `src/api/integrations/chatbot/`:
- Extend `BaseChatbotController` and `BaseChatbotService`
- Implement required methods: `triggerBot`, session management
- Add Prisma model for session/config storage
- Register in `server.module.ts` and route in `chatbot.router.ts`

### Adding a new event destination
Follow the pattern in `src/api/integrations/event/`:
- Implement event publishing logic
- Add configuration to `env.config.ts`
- Register in `EventManager` (`event.manager.ts`)
- Configure event types to forward in environment

### Modifying message handling
- Core logic in `WAMonitoringService.messageHandle()`
- Integration triggers in `findBotByTrigger` utility
- Database saves controlled by `DATABASE_SAVE_DATA_*` flags
- Event emissions through `eventManager`

### Database schema changes
```bash
# 1. Edit prisma/schema.prisma
# 2. Generate new Prisma client
npm run db:generate
# 3. Push to database (no migration files created)
npm run db:deploy
# 4. Restart application to use new schema
```

## Environment Configuration

Key environment variables (see `.env.example` for complete list):
- `SERVER_TYPE` - http or https
- `SERVER_URL` - Base URL for webhook callbacks
- `DATABASE_CONNECTION_URI` - PostgreSQL connection string
- `DATABASE_CONNECTION_CLIENT_NAME` - Isolate multiple API instances on same DB
- `DATABASE_SAVE_DATA_*` - Fine-grained control over what data is persisted
- `AUTHENTICATION_API_KEY` - Global API key for authentication
- Integration-specific config for Chatwoot, Typebot, Dify, OpenAI, etc.
- Event destination config for RabbitMQ, SQS, WebSocket, Webhook

## License Considerations

Apache 2.0 with additional conditions:
1. Cannot remove/modify LOGO or copyright in frontend components
2. Must display notification that Evolution API is being used in any derived project
3. Contact contato@evolution-api.com for commercial licensing if required
