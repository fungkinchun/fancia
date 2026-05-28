# Fancia

Fancia is a social platform connecting people with shared interests for offline, in-person group gatherings and community building.

## README

This project is a backend-focused demonstration of a microservice REST API running on Kubernetes (AWS).

Table of contents

- Overview
- List of microservices (OpenAPI)
- Demo (quick start)
- Technical discussion
- Kafka example (producer / consumer)

## Overview

A collection of microservices that provide authentication, user management, interest groups, and event management. The infrastructure is maintained in
[fancia-infra](https://github.com/fungkinchun/fancia-infra) and [fancia-helm](https://github.com/fungkinchun/fancia-helm). The architecture favors small, focused services with explicit APIs and asynchronous messaging for state synchronization when needed.

There are  notes for individual project. Please check it out.

### List of microservices (OpenAPI)

- auth — [Swagger UI](https://api.fancia.co.uk/auth/swagger-ui/index.html)
- common — [Swagger UI](https://api.fancia.co.uk/common/swagger-ui/index.html)
- user — [Swagger UI](https://api.fancia.co.uk/user/swagger-ui/index.html)
- interestgroup — [Swagger UI](https://api.fancia.co.uk/interestgroup/swagger-ui/index.html)
- event — [Swagger UI](https://api.fancia.co.uk/event/swagger-ui/index.html)

### Infrastructure as code

- [fancia-infra](https://github.com/fungkinchun/fancia-infra)
- [fancia-infra-pipeline](https://github.com/fungkinchun/fancia-infra-pipeline)
- [fancia-helm](https://github.com/fungkinchun/fancia-helm)

## Demo

Scenario: David Smith wants to host a New Year's Eve event on Fancia and invites his friend Olivia Taylor to join.

1. (David) Create a user

    ```bash
    curl -X POST "https://api.fancia.co.uk/user/api/users" \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -d '{
        "email": "david.smith@gmail.com",
        "password": "david-password",
        "confirmPassword": "david-password",
        "firstName": "David",
        "lastName": "Smith"
    }'
    ```

    Server response:

    ```json
    {
        "id": "67ef441d-6f2e-4375-8b3b-da53eb0060f7",
        "role": "USER",
        "firstName": "David",
        "lastName": "Smith",
        "email": "david.smith@gmail.com",
        "profileImageUrl": "",
        "connectedAccounts": [],
        "authorities": [
            "ROLE_USER"
        ]
    }
    ```

2. (David) Login: [Authorize](https://api.fancia.co.uk/auth/oauth2/authorize)

    Navigate to the following URL in your browser (this is an interactive endpoint):

    ```bash
    https://api.fancia.co.uk/auth/oauth2/authorize?client_id=oidc-client&response_type=code&redirect_uri=http%3A%2F%2Ffancia.co.uk%2Flogin%2Foauth2%2Fcode%2Foidc-client&code_challenge=<your-code-challenge>&code_challenge_method=S256&scope=openid%20email%20profile%20client.create%20client.read
    ```

    This is interactive endpoint, use browser instead of cli

    **PKCE (Proof Key for Code Exchange)**: This is a security mechanism for OAuth 2.0 authorization code flow. You can generate a code verifier/challenge pair using the following command:

    ```bash
    python3 -c "import os, base64, hashlib; v = base64.urlsafe_b64encode(os.urandom(32)).decode().rstrip('='); c = base64.urlsafe_b64encode(hashlib.sha256(v.encode()).digest()).decode().rstrip('='); print(f'\nCode Verifier:  {v}\nCode Challenge: {c}\n')"
    ```

    In this example:

    ```bash
    Code Verifier:  tCBFHPCapt6KonVQJr1peENldIpDKgkit8vBRGjvyII
    Code Challenge: w4Rd9ZVjmm1MZrFXRH0JmbtpOF8SAP6EaUYkOTniY74
    ```

    ```bash
     https://api.fancia.co.uk/auth/oauth2/authorize?client_id=oidc-client&response_type=code&redirect_uri=https%3A%2F%2Ffancia.co.uk%2Flogin%2Foauth2%2Fcode%2Foidc-client&code_challenge=w4Rd9ZVjmm1MZrFXRH0JmbtpOF8SAP6EaUYkOTniY74&code_challenge_method=S256&scope=openid%20email%20profile%20client.create%20client.read
    ```

    Server response:

    In normal cases, you will be redirected to the URI you specified. Here we just extract the authorization code:

    ```
    https://api.fancia.co.uk/login/oauth2/code/oidc-client?code=Sdy6N9PbMkgGQLOJODLbnbinPcEnj89Hg8ncLsZEt_snhRxbUH_af6la_p103XALCDiABZ_lxt3mLU1VWmiacGteGgbiL6-qd7tdegeZ5_bYooU7Fc_efmVl-Kn6-HFt
    ```

3. (David) Get token: [Token endpoint](https://api.fancia.co.uk/auth/oauth2/token)

    ```bash
    curl -X POST \
    --url "https://api.fancia.co.uk/auth/oauth2/token" \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "grant_type=authorization_code" \
    --data-urlencode "client_id=oidc-client" \
    --data-urlencode "code=<Sdy6N9PbMkgGQLOJODLbnbinPcEnj89Hg8ncLsZEt_snhRxbUH_af6la_p103XALCDiABZ_lxt3mLU1VWmiacGteGgbiL6-qd7tdegeZ5_bYooU7Fc_efmVl-Kn6-HFt" \
    --data-urlencode "code_verifier=tCBFHPCapt6KonVQJr1peENldIpDKgkit8vBRGjvyII" \
    --data-urlencode "redirect_uri=https://api.fancia.co.uk/login/oauth2/code/oidc-client"
    ```

    Server response:

    ```json
    {
        "access_token":"eyJraWQiOiI1MTkxOGU5Mi03NzQ0LTRiYmItYjU4YS04NjM0N2Q1NTczMmMiLCJhbGciOiJSUzI1NiJ9...",
        "scope":"openid profile client.create client.read email","id_token":"eyJraWQiOiI1MTkxOGU5Mi03NzQ0LTRiYmItYjU4YS04NjM0N2Q1NTczMmMiLCJhbGciOiJSUzI1NiJ9...",
        "token_type":"Bearer",
        "expires_in":604799
    }
    ```

    The decoded token:

    ```json
    {
        "sub": "david.smith@gmail.com",
        "aud": "oidc-client",
        "nbf": 1777134755,
        "scope": [
            "openid",
            "profile",
            "client.create",
            "client.read",
            "email"
        ],
        "iss": "https://api.fancia.co.uk/auth",
        "name": "David Smith",
        "exp": 1777739555,
        "iat": 1777134755,
        "userId": "71745f5a-cdd9-486d-ad4e-98916472e0db",
        "jti": "02736be2-a5d3-4ed0-93e5-acdaaaba70c9",
        "authorities": [
            "ROLE_USER",
            "FACTOR_PASSWORD"
        ],
        "email": "david.smith@gmail.com"
    }
    ```

4. (David) Create an interest group: [Create interest group](https://api.fancia.co.uk/interestgroup/api/interest-groups)

    ```bash
    curl -X POST "https://api.fancia.co.uk/interestgroup/api/interest-groups" \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer eyJraWQiOiI1MTkxOGU5Mi03NzQ0LTRiYmItYjU4YS04NjM0N2Q1NTczMmMiLCJhbGciOiJSUzI1NiJ9..." \
      -d '{
        "name": "London celebrations",
        "description": "Informal meetups and parties around London",
        "tags": ["london","party","celebration"]
      }'
    ```

    Server response:

    ```json
    {
        "content": [
        {
            "id": "79d3b1e2-4f5a-4b6c-8d9e-0f1a2b3c4d5e",
            "name": "London celebrations",
            "description": "Informal meetups and parties around London",
            "createdBy": "7f29c48d-61a3-4e50-9b62-d9f7a831c40b",
            "createdAt": "2025-12-30T14:30:25.123456",
            "tags": ["london","party","celebration"]
        }]
    }
    ```

5. (David) Create an event: [Create event](https://api.fancia.co.uk/event/api/events)

    ```bash
    curl -X POST "https://api.fancia.co.uk/event/api/events" \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer <david-bearer-token>" \
      -d '{
        "name": "New Year's Eve 2026"
        "description": "New Year's Eve Celebrations near London's Eye"
        "startTime": "2026-01-01T00:00:00",
        "duration": "PT1H30M",
        "interestGroupId": "79d3b1e2-4f5a-4b6c-8d9e-0f1a2b3c4d5e",
        "tags": ["london", "nye", "londonEye"]
      }'
    ```

    ```bash
    {
        "content": [
        {
            "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
            "name": "New Year's Eve 2026"
            "description": "Informal meetups and parties around London",
            "createdBy": "7f29c48d-61a3-4e50-9b62-d9f7a831c40b",
            "createdAt": "2025-12-30T14:30:25.123456",
            "tags": ["london", "nye", "londonEye"]
        }]
    }
    ```

6. (Olivia) Create a user (Repeat step 1)

    ```bash
    curl -X POST "https://api.fancia.co.uk/user/api/users" \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -d '{
        "email": "olivia.taylor@gmail.com",
        "password": "olivia-password"
        "confirmPassword": "olivia-password",
        "firstName": "Olivia",
        "lastName": "Taylor"
    }'
    ```

    Server response:

    ```json
    {
        "id": "9b3e1a74-d2c5-4f80-b6a1-3e4729f5d082",
        "role": "USER",
        "firstName": "Olivia",
        "lastName": "Taylor",
        "email": "olivia.taylor@gmail.com"
    }
    ```

7. (Olivia) Get authorization token (Repeat step 2 & 3)

8. (Olivia) Join an interest group: [Join interest group](https://api.fancia.co.uk/interestgroup/api/interest-groups/{interestGroupId}/memberships)

    ```bash
    curl -X POST "https://api.fancia.co.uk/interestgroup/api/interest-groups/79d3b1e2-4f5a-4b6c-8d9e-0f1a2b3c4d5e/memberships" \
     -H "Content-Type: application/json" \
    -H "Authorization: Bearer <olivia-bearer-token>" \
    -d '{
        "payload": ""
    }'
    ```

9. (Olivia) Join a reservation for an event: `POST /event/api/events/{eventId}/reservations`

    ```bash
    curl -X POST "https://api.fancia.co.uk/event/api/events/f47ac10b-58cc-4372-a567-0e02b2c3d479/reservations" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer <olivia-bearer-token>" \
    -d '{
        "guest": 0,
        "payload": ""
    }'

10. (Host) Approve a reservation: `PATCH /event/api/events/{eventId}/users/{userId}/reservations`

    ```bash
    curl -X PATCH "https://api.fancia.co.uk//event/api/events/f47ac10b-58cc-4372-a567-0e02b2c3d479/users/9b3e1a74-d2c5-4f80-b6a1-3e4729f5d082/reservations" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer <david-bearer-token>" \
    -d '{
        "guest": 0,
        "payload": "",
        "status": "ACCEPTED"
    }'
    ```

11. (Olivia) Look for event participants: `GET /event/api/events/{eventId}/participants`

    ```bash
    curl -X GET "https://api.fancia.com/events/f47ac10b-58cc-4372-a567-0e02b2c3d479/participants?page=0&size=20" \
    -H "Accept: application/json"
    ```

    Server response:

    ```json
    {
        "content": [
        {
            "userId": "7f29c48d-61a3-4e50-9b62-d9f7a831c40b",
            "role": "HOST"
        },
        {
            "userId": "9b3e1a74-d2c5-4f80-b6a1-3e4729f5d082",
            "role": "Guest"
        }]
    }
    ```

Notes:

- Replace the `{eventId}` and `{userId}` placeholders with the actual IDs returned by the API.
- To register OIDC clients, see the [auth](https://github.com/fungkinchun/fancia-backend-auth) service repository and its README for registration endpoints and examples; pay attention to redirect URIs, required scopes, PKCE for public clients, and secure storage of client secrets.
- Most list endpoints are using Fuzzy search

## Technical discussion

This section summarizes key architectural decisions and operational practices used across the platform.

### Library sharing

The codebase is Kotlin-first and shares common models and utilities via a set of shared repositories published as artifacts. Consumers include service modules that import the shared artifacts via Gradle/Maven.

- [shared-common](https://github.com/fungkinchun/fancia-backend-shared-common)
- [shared-user](https://github.com/fungkinchun/fancia-backend-shared-user)
- [shared-interestgroup](https://github.com/fungkinchun/fancia-backend-shared-interestgroup)
- [shared-event](https://github.com/fungkinchun/fancia-backend-shared-event)

Notes:

- Publish shared modules to an internal artifact repository on CodeArtifact for stable consumption.
- Keep shared DTOs and serialization stable and versioned to avoid breaking consumers.

### Inter-service communications

The system uses two primary communication patterns:

#### Synchronous HTTP between services (OpenFeign)

OpenFeign is used for simple request/response calls between services (e.g., lookup operations, authorization checks).

One example is when marking a user for deletion, the system checks whether the user still holds any administrative memberships; if any are found, deletion is rejected to prevent orphaning resources and preserve data integrity.

```kotlin
package com.fancia.backend.user.external

import com.fancia.backend.shared.interestgroup.core.dto.InterestGroupMembershipResponse
import com.fancia.backend.shared.interestgroup.core.enums.InterestGroupRole
import com.fancia.backend.user.config.FeignConfig
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.data.domain.Page
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestParam
import java.util.*

@FeignClient(name = "interestgroup-service", path = "/api", configuration = [FeignConfig::class])
interface InterestGroupServiceClient {
    @GetMapping("/interest-groups/users/{userId}/memberships")
    fun getInterestGroupMembership(
        @RequestParam("userId") userId: UUID,
        @RequestParam("role") role: InterestGroupRole = InterestGroupRole.ADMIN,
        @RequestParam(value = "page", required = false) page: Int = 0,
        @RequestParam(value = "size", required = false) size: Int = 20
    ): Page<InterestGroupMembershipResponse>
}
```

#### Asynchronous messaging (Kafka)

Kafka is used for propagating domain events (deletions, updates) and for decoupling services that should react to changes eventually.

##### Key Concepts

- Bootstrap Servers: Initial connection points that clients use to discover the full cluster topology
- Partitions: Units of scalability and ordering; messages with the same key are routed to the same partition to maintain order
- Replication: Kafka's durability mechanism using a leader-follower pattern where each partition has one leader and multiple followers
- Pull-based Architecture: Consumers actively pull messages from brokers, and follower replicas pull data from partition leaders

##### Idempotent Producers

Idempotence is enabled by default with `enable.idempotence: true`. An idempotent producer ensures that no matter how many times you send the same message, the result is the same as sending it once.

Without idempotence in Kafka:

- If a message fails to be acknowledged (due to network issues, timeouts, etc.), the producer will retry
- This can result in duplicate messages being written to the topic

##### How Idempotent Producers Work

Kafka's idempotent producer solves the duplication problem by guaranteeing that each message is written only once, even if retried multiple times. It uses these mechanisms:

- Producer ID (PID): Each producer instance receives a unique identifier from the broker
- Sequence Numbers: Every message gets a sequence number per partition  
- Broker-side Deduplication: The broker tracks the (PID + Sequence Number) combination and ignores duplicates

This enables safe retries without creating duplicate messages.

##### Configuration Requirements

For idempotent producers to work correctly, the following settings are required:

- `acks=all` - Wait for acknowledgment from all in-sync replicas
- `retries > 0` - Enable retry attempts for failed sends
- `max.in.flight.requests.per.connection ≤ 5` - Limit concurrent requests to maintain ordering

#### Example: user deletion (producer)

```kotlin
package com.fancia.backend.user.core.message

import com.fancia.backend.shared.user.core.message.UserDeletedEvent
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.stereotype.Service
import java.util.*

@Service
class UserProducer(
    private val kafkaTemplate: KafkaTemplate<UUID, Any>
) {
    fun publishUserDeleted(event: UserDeletedEvent) {
        kafkaTemplate.send("users", event.id, event)
            .whenComplete { result, ex -> }
    }
}
```

#### Example: user deletion (consumer)

```kotlin
package com.fancia.backend.interestgroup.core.message

import com.fancia.backend.interestgroup.core.service.InterestGroupMembershipService
import com.fancia.backend.shared.user.core.message.UserDeletedEvent
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.stereotype.Component

@Component
class UserConsumer(
    private val interestGroupMembershipService: InterestGroupMembershipService
) {
    @KafkaListener(topics = ["users"], groupId = "deletion")
    fun onUserDeleted(event: UserDeletedEvent) {
        interestGroupMembershipService.removeMemberFromAllGroups(event.id)
    }
}
```

### Testing

#### Integration tests

Kotest is used as the primary test framework. External HTTP dependencies are mocked with WireMock running inside a Testcontainers WireMockContainer. Integration tests use Testcontainers to run databases, Kafka, and other dependencies locally in CI. Controller-level requests are exercised with Spring's MockMvc.

Typical test class pattern used in the repo:

- Annotate with `@SpringBootTest`, `@AutoConfigureMockMvc`, `@Testcontainers`, and `@Import(TestConfig::class)`.
- Inject `WireMockContainer` and configure the WireMock client in a test setup (for example, `configureFor(wiremock.host, wiremock.getMappedPort(8080))`).
- Use a `JsonMapper` for serialization/deserialization and a mapper (e.g., `UserMapper`) to convert shared DTOs to domain beans.

Summary of example test: `UserControllerIntegrationTest`

- Purpose: an end-to-end controller integration test that verifies user creation, persistence, external service interaction, and guarded deletion behavior.

- Flow and assertions:
  1. POST `/api/users` via `MockMvc` with a JSON payload (email, password, confirmPassword, firstName, lastName).
  2. Assert the response is 200 OK and that the JSON body contains the expected `email` and a non-null `id`.
  3. Convert the response JSON to a domain `User` using `JsonMapper` and `UserMapper`, then verify persistence with `userRepository.findByIdOrNull`.
  4. Stub the interestgroup service's membership endpoint using WireMock to return an ADMIN membership page for the created user.
  5. Attempt DELETE `/api/users/{id}` with a JWT that contains the `userId` claim; expect `400 Bad Request` (deletion is prevented due to existing memberships).

- Helper used in the test:
  - `ResultActionsDsl.toUser(jsonMapper, userMapper)` extension converts the MockMvc response into a `User` domain object by reading `UserResponse` and mapping it.

Best practices and tips for integration tests in this repo:

- Reset WireMock stubs between tests (e.g., `wiremock.resetAll()`) to avoid cross-test interference.
- Use Testcontainers' reusable containers in CI where possible; ensure proper startup timeouts and container health checks.
- Clean or seed the database between tests to keep tests isolated and deterministic.
- Make external service stubs return representative paginated responses and edge cases (empty results, errors) to exercise failure modes.

### Monitoring and observability

This project uses Prometheus, Grafana, Loki, and Alloy to provide metrics, dashboards, and centralized logs. Alloy is typically deployed as a logging sidecar that captures container stdout/stderr and forwards logs to Loki through the Kubernetes logging pipeline. Prometheus scrapes application metrics (for example, via /actuator/prometheus) and Grafana is used to visualize both metrics and Loki queries so you can correlate telemetry with logs.

API errors are commonly handled by a `ValidationHandler` present in each service. A typical error response looks like:

```json
{
  "detail": "Validation failed: User with this email already exists",
  "instance": "/user/api/users",
  "status": 400,
  "title": "Validation Error",
  "type": "fancia.co.uk/user/api",
  "errorCode": "VALIDATION_ERROR"
}
```

To locate related logs in Grafana (Loki) use a query such as:

```loki
{namespace="fancia-prod", service_name="user"} |= "Validation Error"
```

This query returns application logs from the user service in the production namespace that include the "Validation Error" title. For more effective troubleshooting, augment queries with labels such as pod, container, or request id and use Grafana's Explore view to correlate logs and metrics.

#### Kubernetes cluster logs

Cluster-level logs and Kubernetes events can be viewed in Grafana dashboards or via cloud provider consoles and tools. In AWS, enable control plane logging and aggregate node and kubelet logs to a central store for auditing and incident investigation.

#### Infrastructure logs

Infrastructure events and API-level audit trails are available in AWS CloudTrail, while CloudWatch collects logs and metrics emitted by AWS services and custom agents. Use CloudTrail for who-did-what auditing and CloudWatch for operational telemetry; together they provide comprehensive visibility for security and incident response.

### Challenges and future improvements

#### Working with MapStruct

MapStruct can be awkward in Kotlin projects because MapStruct expects Java-style setters and constructors. Kotlin data classes and val properties sometimes prevent MapStruct from generating direct mappings. To reduce friction, configure kapt and MapStruct correctly, use `@Mapper(componentModel = "spring")`, provide no-arg constructors or `@Default` values for targets when needed, or use `var` properties for mapped targets. For complex mapping scenarios, consider Kotlin-native alternatives such as Konvert or manual mappers that avoid annotation processing and produce more idiomatic Kotlin code.

#### Caching

Introduce Redis for query caching or session storage where appropriate, but pay careful attention to cache invalidation, key design, and TTLs. To maintain consistency between caches and the source of truth, consider event-driven invalidation patterns using Kafka domain events so services can react to state changes and refresh or evict stale entries.

#### Adoption of Elasticsearch

This project currently uses PostgreSQL's `pg_trgm` extension (trigram matching) for fuzzy search. It is a lightweight, low-overhead option that works well for typo-tolerant lookups and LIKE-based queries without introducing extra infrastructure. For requirements that need richer full-text features, custom analyzers, complex relevance tuning, or higher-scale search performance, consider introducing Elasticsearch.
