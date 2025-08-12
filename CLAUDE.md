# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## OpenTelemetry Demo - Microservice E-Commerce Application

This is a production-level demonstration application consisting of 13 microservices written in different languages to showcase OpenTelemetry instrumentation, distributed tracing, metrics, and observability best practices.

## Core Development Commands

### Build and Run
```bash
# Start the full demo (all services and observability stack)
make start

# Start minimal version (fewer services, less resources)
make start-minimal

# Stop all services and clean up volumes
make stop

# Build all Docker images
make build

# Build for multiple platforms (AMD64/ARM64)
make create-multiplatform-builder
make build-multiplatform
```

### Development Workflow
```bash
# Restart a specific service (service names match src/ directories)
make restart service=frontend
make restart service=cart

# Rebuild and restart a service after code changes
make redeploy service=checkout

# Run quality checks (lint, license, links)
make check

# Auto-fix issues where possible
make fix
```

### Testing
```bash
# Run all tests (frontend Cypress + trace-based tests)
make run-tests

# Run trace-based tests only
make run-tracetesting

# Run tests for specific services
make run-tracetesting SERVICES_TO_TEST=frontend

# Run frontend tests locally
cd src/frontend && npm test
cd src/frontend && npm run cy:open  # Interactive Cypress
```

## High-Level Architecture

### Service Communication Pattern
```
Frontend (Next.js) 
    → gRPC → Cart Service (C#/.NET)
    → gRPC → Product Catalog (Go)
    → gRPC → Currency Service (C++)
    → gRPC → Ad Service (Java)
    → gRPC → Recommendation Service (Python)
    
Frontend → Checkout Service (Go)
    → Kafka → Fraud Detection (Kotlin)
    → Kafka → Accounting Service (C#/.NET)
    → gRPC → Shipping Service (Rust)
    → gRPC → Payment Service (Node.js)
    → gRPC → Email Service (Ruby)
    → gRPC → Quote Service (PHP)
```

### Data Stores
- **Valkey (Redis)**: Shopping cart session storage
- **PostgreSQL**: Accounting service persistence
- **Kafka**: Event streaming for asynchronous workflows

### Key Architectural Patterns

1. **Protocol Buffers**: All inter-service communication uses gRPC with shared protobuf definitions in `pb/demo.proto`. Each service generates language-specific clients from this shared definition.

2. **OpenTelemetry Instrumentation**: Every service implements OpenTelemetry SDK with consistent:
   - Resource attributes (service.name, service.version)
   - Trace context propagation across service boundaries
   - Metrics collection (latency, error rates, custom business metrics)
   - Structured logging with trace correlation

3. **Feature Flags**: Using OpenFeature with Flagd for dynamic configuration. Feature flags control:
   - Service behavior (e.g., payment processing delays)
   - Failure injection for chaos testing
   - A/B testing scenarios

4. **Event-Driven Architecture**: Checkout flow triggers Kafka events consumed by:
   - Fraud Detection Service for transaction analysis
   - Accounting Service for financial recording

## Service-Specific Development

### Frontend (Next.js/TypeScript)
```bash
# Local development with hot reload
docker compose run --service-ports -e NODE_ENV=development \
  --volume $(pwd)/src/frontend:/app \
  --volume $(pwd)/pb:/app/pb \
  --user node --entrypoint sh frontend
# Inside container: npm run dev
```

### Backend Services
Most backend services follow similar patterns:
- Environment configuration via `.env` files
- Health check endpoints at `/health`
- gRPC server on specific ports (see `.env` for mappings)
- OpenTelemetry auto-instrumentation where available

## Testing Strategy

### Trace-Based Testing (Primary)
Located in `test/tracetesting/`, uses Tracetest to validate:
- Service interactions produce expected traces
- Span attributes match requirements
- Service dependencies are correctly instrumented
- Performance characteristics meet SLOs

### E2E Testing
Frontend Cypress tests in `src/frontend/cypress/` validate:
- User workflows (browse, add to cart, checkout)
- UI component behavior
- API integration

## Important Configuration Files

- **`.env`**: Base configuration for all services
- **`.env.override`**: Local overrides (gitignored)
- **`.env.arm64`**: ARM64/M-series Mac specific settings
- **`docker-compose.yml`**: Full stack orchestration
- **`docker-compose.minimal.yml`**: Lightweight development
- **`pb/demo.proto`**: Shared gRPC service definitions

## Observability Access Points

When running locally, access observability tools at:
- **Application**: http://localhost:8080/
- **Jaeger UI**: http://localhost:8080/jaeger/ui/
- **Grafana**: http://localhost:8080/grafana/
- **Feature Flags**: http://localhost:8080/feature/
- **Load Generator**: http://localhost:8080/loadgen/

## Common Development Tasks

### Adding a New Feature
1. Update `pb/demo.proto` if API changes needed
2. Regenerate protobuf code: `make generate-protobuf`
3. Implement feature in target service(s)
4. Add OpenTelemetry instrumentation
5. Create trace-based tests in `test/tracetesting/`
6. Update frontend if user-facing

### Debugging Service Issues
1. Check service logs: `docker compose logs <service-name>`
2. View traces in Jaeger for request flow
3. Check Grafana dashboards for metrics
4. Use feature flags to isolate issues
5. Run trace-based tests for validation

### Performance Optimization
1. Use Grafana to identify bottlenecks
2. Analyze traces for slow spans
3. Check resource limits in docker-compose.yml
4. Profile service-specific code
5. Validate improvements with load generator

## Key Conventions

- **Service Naming**: Consistent `OTEL_SERVICE_NAME` environment variable
- **Port Allocation**: Each service has dedicated ports (see `.env`)
- **Resource Attributes**: Standard OpenTelemetry semantic conventions
- **Error Handling**: Propagate errors with proper gRPC status codes
- **Logging**: Structured JSON logging with trace context
- **Health Checks**: All services implement `/health` endpoints

## Multi-Language Considerations

Each service uses language-specific best practices:
- **Java/Kotlin**: Gradle build, OpenTelemetry Java Agent
- **Go**: Native modules, manual instrumentation
- **.NET/C#**: Entity Framework, ASP.NET Core patterns
- **Python**: gRPC with manual instrumentation
- **Node.js**: Auto-instrumentation with require hooks
- **Rust**: Tonic for gRPC, manual instrumentation
- **Ruby**: Sinatra framework with OpenTelemetry SDK
- **PHP**: Slim framework with OpenTelemetry extension

## Environment Variables

Critical environment variables used across services:
- `OTEL_EXPORTER_OTLP_ENDPOINT`: OpenTelemetry Collector endpoint
- `OTEL_SERVICE_NAME`: Service identifier for traces/metrics
- `FEATURE_FLAG_SERVICE`: Flagd service URL
- `*_SERVICE_ADDR`: Inter-service communication addresses

## Troubleshooting

### Service Won't Start
- Check port conflicts: `docker ps`
- Verify environment variables: `docker compose config`
- Check service dependencies are running
- Review logs: `docker compose logs <service>`

### Traces Not Appearing
- Verify OTEL Collector is running
- Check `OTEL_EXPORTER_OTLP_ENDPOINT` configuration
- Ensure service has OpenTelemetry SDK initialized
- Look for errors in service logs

### Build Failures
- Clear Docker cache: `docker system prune`
- Rebuild specific service: `make redeploy service=<name>`
- Check Docker resource limits
- For M-series Macs, ensure `.env.arm64` is loaded