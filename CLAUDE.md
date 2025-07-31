# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

One Hub is an AI service relay platform based on one-api, designed to proxy and manage multiple AI provider APIs (OpenAI, Anthropic, Gemini, etc.) through a unified interface. It features a Go backend with Gin framework and a React frontend with Material-UI.

## Common Development Commands

### Backend Development (Go)
```bash
# Build and run the full application
task build
task run

# Development with live reload (requires separate terminal for frontend)
go run main.go

# Format and lint code
task fmt        # Combines go mod tidy, gofmt, goimports, and golangci-lint
task lint       # Just formatting and linting

# Build for production
task docker     # Build Docker image with embedded frontend
```

### Frontend Development (React)
```bash
cd web
npm install     # or yarn install

# Development server (connects to backend on port 3000)
npm run dev

# Build for production
npm run build   # Outputs to web/build/

# Linting and formatting
npm run lint
npm run lint:fix
npm run prettier
```

### Build System Integration
```bash
# Build frontend and embed in Go binary (automated by tasks)
hack/scripts/genui.sh <version>

# Clean all build artifacts
task clean
```

### Database Operations
```bash
# The application auto-migrates database schema on startup
# Default: SQLite (one-api.db)
# Production: MySQL/PostgreSQL via SQL_DSN environment variable
```

### Configuration
- Development: `config.yaml` (copy from `config.example.yaml`)
- Production: Environment variables or config file
- Frontend proxy: Development frontend (port 3010) proxies API calls to backend (port 3000)

### Testing
```bash
# Backend tests
go test ./...

# Frontend tests (if available)
cd web && npm test
```

## Architecture Overview

### High-Level System Design

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│   React Frontend │────│   Go Backend     │────│   AI Providers      │
│   (Material-UI)  │    │   (Gin Framework)│    │   (OpenAI/Claude/   │
│   Port: 3010     │    │   Port: 3000     │    │    Gemini/etc.)     │
└─────────────────┘    └──────────────────┘    └─────────────────────┘
                                │
                       ┌────────┴────────┐
                       │   Database      │
                       │ (SQLite/MySQL/  │
                       │  PostgreSQL)    │
                       └─────────────────┘
```

### Core Backend Architecture

**Main Application Flow** (`main.go`):
1. Configuration initialization (`config.InitConf()`)
2. Database setup and migration (`model.SetupDB()`)
3. Cache and Redis initialization
4. Provider and relay system initialization
5. HTTP server startup with Gin router

**Key System Components**:

1. **Router System** (`router/`):
   - `SetApiRouter()`: Admin/management APIs
   - `SetRelayRouter()`: AI provider relay endpoints
   - `SetDashboardRouter()`: Frontend dashboard APIs
   - `SetWebRouter()`: Static file serving (embedded frontend)
   - `SetMcpRouter()`: MCP (Model Context Protocol) server integration

2. **Relay Architecture** (`relay/`):
   - **Path2Relay()**: Maps incoming requests to appropriate relay handlers
   - **RelayHandler()**: Core request processing with retry logic and quota management
   - **Provider Interface**: Standardized interface for all AI providers
   - **Token Management**: Prompt/completion token calculation and quota enforcement
   - **Error Handling**: Automatic retry with different providers on failure

3. **Provider System** (`providers/`):
   - **Base Interface** (`providers/base/interface.go`): Defines standard provider contract
   - **Individual Providers**: Each AI service (OpenAI, Anthropic, etc.) implements the interface
   - **Model Mapping**: Translates between user-requested models and provider-specific models
   - **Request/Response Transformation**: Handles format differences between providers

4. **Channel Management** (`model/channel.go`):
   - **Load Balancing**: Distributes requests across multiple provider instances
   - **Health Monitoring**: Tracks provider availability and performance
   - **Cooldown System**: Temporarily disables failed providers
   - **Priority/Weight System**: Routes requests based on configured priorities

5. **Database Layer** (`model/`):
   - **GORM ORM**: Database abstraction with auto-migration
   - **Models**: User, Channel, Token, Log, Pricing, etc.
   - **Caching**: Memory and Redis caching for frequently accessed data
   - **Batch Operations**: Optimized bulk updates for high-throughput scenarios

### Frontend Architecture

**Technology Stack**:
- **React 18** with functional components and hooks
- **Material-UI v5** for component library
- **Vite** for build tooling and development server
- **React Router v6** for client-side routing
- **Redux** for state management
- **i18next** for internationalization

**Key Directories**:
- `src/views/`: Main application pages (Dashboard, Channel, User management, etc.)
- `src/components/`: Reusable UI components
- `src/hooks/`: Custom React hooks for API calls and state
- `src/contexts/`: React context providers for global state
- `src/utils/`: API client and utility functions

**Build Process**:
1. Frontend builds to `web/build/`
2. Go embed directive includes frontend in binary
3. Router serves static files from embedded filesystem
4. Development uses Vite dev server with API proxy

### Request Flow Architecture

**Typical API Request Flow**:
1. **Client Request** → Frontend or API client
2. **Authentication** → Middleware validates API key/JWT
3. **Rate Limiting** → Request counting and throttling
4. **Model Resolution** → Map user model name to provider model
5. **Provider Selection** → Choose available provider based on model/load balancing
6. **Request Transformation** → Convert to provider-specific format
7. **Relay Execution** → Send request to AI provider
8. **Response Processing** → Transform response back to OpenAI format
9. **Usage Tracking** → Log tokens consumed and update quotas
10. **Client Response** → Return formatted response

**Retry Logic**:
- Automatic retry on failure with different providers
- Cooldown system for temporarily failed providers
- Configurable retry attempts and timeout
- Circuit breaker pattern for provider health

**Streaming Support**:
- Server-Sent Events (SSE) for real-time responses
- Proper connection management and cleanup
- Token counting during stream processing

### Database Schema

**Core Entities**:
- **Users**: Authentication, quotas, groups, billing
- **Channels**: Provider configurations, API keys, model mappings
- **Tokens**: API keys for users with scopes and limitations
- **Logs**: Request/response tracking, usage analytics
- **Models**: Pricing, capabilities, provider mappings
- **Orders**: Payment processing and billing records

**Key Relationships**:
- Users have multiple Tokens
- Channels belong to specific Providers
- Logs track usage per User/Channel/Model
- Pricing varies by Model and User Group

## Provider Integration Pattern

When adding new AI providers:

1. **Create Provider Package** (`providers/newprovider/`):
   ```go
   type NewProvider struct {
       base.BaseProvider
       // Provider-specific fields
   }
   
   func (p *NewProvider) CreateChatCompletion(request *types.ChatCompletionRequest) (*types.ChatCompletionResponse, *types.OpenAIErrorWithStatusCode)
   ```

2. **Implement Required Interfaces**:
   - `ChatInterface` for chat completions
   - `EmbeddingsInterface` for embeddings
   - `ImageGenerationsInterface` for image generation
   - etc.

3. **Register Provider** (`providers/providers.go`):
   ```go
   func GetProvider(channel *model.Channel) (base.ProviderInterface, error) {
       switch channel.Type {
       case types.NewProvider:
           return newprovider.CreateNewProvider(channel)
       }
   }
   ```

4. **Add Frontend Configuration** (`web/src/views/Channel/type/`):
   - Configuration UI components
   - Validation rules
   - Model lists and capabilities

## Security Considerations

**API Security**:
- JWT-based session management
- API key authentication with scopes
- Rate limiting per IP and user
- Request/response validation

**Provider Security**:
- API keys stored encrypted
- Proxy support for network isolation  
- Request sanitization
- Response filtering

**Data Privacy**:
- Request logging configurable
- No sensitive data in logs by default
- GDPR compliance features

## Performance Optimization

**Caching Strategy**:
- Memory cache for frequently accessed data
- Redis for distributed caching
- Provider response caching (when appropriate)
- Database query optimization

**Scaling Patterns**:
- Master/Slave deployment for horizontal scaling
- Database connection pooling
- Provider request queuing
- Batch processing for logs and analytics

## Deployment Patterns

**Single Instance** (Docker):
```bash
docker run -p 3000:3000 -v ./data:/data martialbe/one-api:latest
```

**Multi-Instance** (Docker Compose):
- Master node serves frontend and manages database
- Slave nodes handle API relay requests
- Shared MySQL/PostgreSQL database
- Redis for cross-instance caching

**Configuration Management**:
- Environment variables for production
- YAML config file for development
- Database-stored runtime configuration
- Admin UI for dynamic configuration changes

## Integration Points

**MCP (Model Context Protocol)** Server:
- Enables Claude and other AI assistants to access One Hub functionality
- Provides tools for model management, usage tracking, and configuration
- Located in `mcp/` directory with tool implementations

**Monitoring and Metrics**:
- Prometheus metrics export
- Request/response logging
- Provider health monitoring
- Usage analytics and billing reports

**Payment Processing**:
- Support for Stripe, Alipay, WeChat Pay
- Automated billing and invoice generation
- Credit/quota management system
- Payment webhook processing

## Testing Strategy

**Backend Testing**:
- Unit tests for core business logic
- Integration tests for provider implementations
- Database migration testing
- API endpoint testing

**Frontend Testing**:
- Component unit tests
- Integration tests for API interactions
- E2E testing for critical user flows
- Accessibility testing

**Provider Testing**:
- Mock provider responses for development
- Provider health check automation
- Load testing for relay performance
- Failover scenario testing