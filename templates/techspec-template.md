# Technical Specification Template

## Executive Summary

[Provide a brief technical overview of the solution approach. Summarize the main architectural decisions and implementation strategy in 1–2 paragraphs.]

## System Architecture

### Component Overview

[Briefly describe the main components and their responsibilities:

- Component names and primary functions **Be sure to list every new or modified component**
- Main relationships between components
- Overview of the data flow]

## Implementation Design

### Main Interfaces

[Define the main service interfaces (≤20 lines per example):

```go
// Example interface definition
type ServiceName interface {
    MethodName(ctx context.Context, input InputType) (output OutputType, error)
}
```

]

### Data Models

[Define the essential data structures:

- Main domain entities (if applicable)
- Request/response types
- Database schemas (if applicable)]

### API Endpoints

[List API endpoints, if applicable:

- Method and path (e.g., `POST /api/v0/resource`)
- Brief description
- References to request/response formats]

## Integration Points

[Include this section only if the feature requires external integrations:

- External services or APIs
- Authentication requirements
- Error-handling approach]

## Testing Approach

### Unit Tests

[Describe the unit testing strategy:

- Main components to test
- Mocking requirements (external services only)
- Critical test scenarios]

### Integration Tests

[If necessary, describe integration tests:

- Components to test together
- Test data requirements]

### E2E Tests

[If necessary, describe E2E tests:

- Test the frontend together with the backend **using Playwright**]

## Development Sequencing

### Build Order

[Define the implementation sequence:

1. First component/feature (why it comes first)
2. Second component/feature (dependencies)
3. Subsequent components
4. Integration and testing]

### Technical Dependencies

[List any blocking dependencies:

- Required infrastructure
- External service availability]

## Monitoring and Observability

[Define the monitoring approach using the existing infrastructure:

- Metrics to expose (Prometheus format)
- Main logs and log levels
- Integration with existing Grafana dashboards]

## Technical Considerations

### Key Decisions

[Document important technical decisions:

- Chosen approach and rationale
- Trade-offs considered
- Rejected alternatives and why]

### Known Risks

[Identify technical risks:

- Potential challenges
- Mitigation approaches
- Areas requiring research]

### Standard Skills Compliance

[Research the skills in the @.claude/skills folder that fit and apply to this Tech Spec, and list them below:]

### Relevant and Dependent Files

[List relevant and dependent files here]
