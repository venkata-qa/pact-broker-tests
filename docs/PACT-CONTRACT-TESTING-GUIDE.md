# Understanding Pact Contract Testing: A Comprehensive Guide

## Table of Contents
1. [Introduction to Contract Testing](#introduction-to-contract-testing)
2. [What is Pact?](#what-is-pact)
3. [Core Concepts](#core-concepts)
4. [How Pact Works](#how-pact-works)
5. [The Pact Workflow](#the-pact-workflow)
6. [Understanding the Components](#understanding-the-components)
7. [The Pact Broker](#the-pact-broker)
8. [can-i-deploy: Deployment Safety](#can-i-deploy-deployment-safety)
9. [Failure Scenarios and Detection](#failure-scenarios-and-detection)
10. [Best Practices](#best-practices)
11. [Real-World Example Walkthrough](#real-world-example-walkthrough)

---

## Introduction to Contract Testing

### The Problem with Traditional Integration Testing

In microservices architectures, services communicate with each other through APIs. Traditional integration testing has several challenges:

1. **Tight Coupling**: Services must be running simultaneously for tests
2. **Slow Feedback**: Integration tests are slow and require complex setup
3. **Flaky Tests**: Network issues, service availability, and timing problems cause unreliable tests
4. **Expensive**: Requires maintaining test environments with all services running
5. **Breaking Changes**: Changes in one service can break others without early detection

### What is Contract Testing?

**Contract testing** is a methodology that ensures two services (a consumer and a provider) can communicate correctly by verifying that they adhere to a shared "contract" that defines the expected interactions between them.

Think of it like a legal contract:
- The **consumer** defines what it expects from the provider (the contract)
- The **provider** must fulfill those expectations
- If either side breaks the contract, tests fail immediately

### Benefits of Contract Testing

- ✅ **Fast Feedback**: Tests run quickly without requiring all services to be running
- ✅ **Early Detection**: Breaking changes are caught before deployment
- ✅ **Decoupled Testing**: Services can be tested independently
- ✅ **Consumer-Driven**: Consumers define what they need, ensuring providers meet real requirements
- ✅ **Version Management**: Track contract versions and compatibility over time

---

## What is Pact?

**Pact** is a code-first contract testing tool that implements consumer-driven contract testing. It allows you to:

1. Define contracts in code (not manually written documents)
2. Generate contracts automatically from consumer tests
3. Verify contracts against provider implementations
4. Manage contracts centrally through the Pact Broker
5. Prevent breaking changes from being deployed

### Key Characteristics

- **Consumer-Driven**: The consumer defines the contract based on what it actually needs
- **Code-First**: Contracts are defined in test code, not separate documents
- **Language Agnostic**: Works with multiple programming languages (Java, JavaScript, Python, Ruby, Go, etc.)
- **Versioned**: Each contract version is tracked and can be verified independently

---

## Core Concepts

### 1. Consumer

The **consumer** is the service that uses (consumes) an API provided by another service. In our example:
- **Consumer Name**: `users-consumer`
- **Role**: Defines what it expects from the provider
- **Responsibility**: 
  - Write tests that define expected API interactions
  - Generate pact files from these tests
  - Publish pacts to the Pact Broker

**Example from our project:**
```java
@Pact(provider = "users-provider", consumer = "users-consumer")
public RequestResponsePact pactWithAllProperties(final PactDslWithProvider builder) {
    return builder
        .given("user-is-leanne")
        .uponReceiving("Should return all available properties in response body.")
        .path("/users/1")
        .method("GET")
        .willRespondWith()
        .status(200)
        .body("""
            {
              "id":"1",
              "name":"Leanne Graham",
              "username":"leanne",
              "email":"sincere@april.biz"
            }
        """)
        .toPact();
}
```

This code defines:
- **Given**: The provider state ("user-is-leanne")
- **Upon Receiving**: A description of the interaction
- **Request**: GET /users/1
- **Response**: Status 200 with a specific JSON body

### 2. Provider

The **provider** is the service that provides an API to consumers. In our example:
- **Provider Name**: `users-provider`
- **Role**: Must satisfy the contracts defined by consumers
- **Responsibility**:
  - Fetch pacts from the Pact Broker
  - Verify that its implementation matches the contract
  - Publish verification results back to the broker

**Example from our project:**
```java
@Provider("users-provider")
@PactBroker(url = BROKER_URL, providerTags = { "dev" })
class UsersPactTest {
    @BeforeEach
    public void before(PactVerificationContext context) {
        // Set up a mock server that simulates the provider API
        this.server = new WireMockServer(wireMockConfig().port(8081));
        this.server.stubFor(get(urlEqualTo("/users/1"))
            .willReturn(aResponse()
                .withBody(BODY)
                .withHeader("content-type","application/json")));
        this.server.start();
        context.setTarget(HttpTestTarget.fromUrl(new URL(SERVER_URL)));
    }
}
```

### 3. Pact File

A **pact file** is a JSON document that contains the contract between a consumer and provider. It includes:
- Consumer and provider names
- Interactions (requests and expected responses)
- Provider states
- Metadata

**Example pact file structure:**
```json
{
  "consumer": {
    "name": "users-consumer"
  },
  "provider": {
    "name": "users-provider"
  },
  "interactions": [
    {
      "description": "Should return all available properties in response body.",
      "providerState": "user-is-leanne",
      "request": {
        "method": "GET",
        "path": "/users/1"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "body": {
          "id": "1",
          "name": "Leanne Graham",
          "username": "leanne",
          "email": "sincere@april.biz"
        }
      }
    }
  ],
  "metadata": {
    "pactSpecification": {
      "version": "3.0.0"
    }
  }
}
```

### 4. Provider State

A **provider state** is a condition that must be true for a particular interaction to be valid. It allows the same endpoint to return different responses based on the state.

**Example:**
- `"user-is-leanne"`: The provider should have a user with ID 1 named "Leanne Graham"
- `"user-does-not-exist"`: The provider should return a 404 error

In the provider test, you implement state handlers:
```java
@State("user-is-leanne")
public void isLeanne() {
    // Set up the provider state: ensure user with ID 1 exists
    // In a real scenario, this might insert data into a database
}
```

### 5. Verification

**Verification** is the process of checking that a provider's implementation matches the contract defined in a pact. During verification:
1. The provider fetches pacts from the broker
2. For each interaction in the pact:
   - Set up the provider state
   - Make the request to the provider
   - Compare the response with the expected response in the pact
3. Publish verification results back to the broker

---

## How Pact Works

### The Pact Testing Cycle

```
┌─────────────┐
│  Consumer   │
│   Tests     │
└──────┬──────┘
       │
       │ 1. Run consumer tests
       │    (against mock provider)
       ▼
┌─────────────┐
│  Pact File  │
│  Generated  │
└──────┬──────┘
       │
       │ 2. Publish pact
       │    to broker
       ▼
┌─────────────┐
│Pact Broker  │
│  (Central   │
│  Storage)   │
└──────┬──────┘
       │
       │ 3. Fetch pacts
       │    for verification
       ▼
┌─────────────┐
│  Provider   │
│   Tests     │
└──────┬──────┘
       │
       │ 4. Verify against
       │    actual provider
       │    (or mock)
       ▼
┌─────────────┐
│Verification │
│  Results    │
└──────┬──────┘
       │
       │ 5. Publish results
       │    back to broker
       ▼
┌─────────────┐
│Pact Broker  │
│  (Updated   │
│  with       │
│  Results)   │
└─────────────┘
```

### Step-by-Step Process

#### Step 1: Consumer Defines the Contract

The consumer writes a test that:
1. Sets up a mock provider server
2. Defines expected request/response interactions
3. Makes actual HTTP calls to the mock server
4. Asserts the response matches expectations

**What happens:**
- Pact framework generates a mock server based on the contract
- The test runs against this mock server
- If the test passes, a pact file is generated

#### Step 2: Publish Pact to Broker

The consumer publishes the generated pact file to the Pact Broker:
```bash
pact-broker publish "./consumer/build/pacts" \
  --consumer-app-version "1.0.0" \
  --broker-base-url "http://localhost:9292" \
  --tag "main"
```

**What happens:**
- Pact file is uploaded to the broker
- Version information is stored
- Tags are applied (e.g., "main", "dev", "prod")

#### Step 3: Provider Verifies the Contract

The provider:
1. Fetches relevant pacts from the broker
2. For each pact interaction:
   - Sets up the provider state
   - Makes a request to the provider (or mock)
   - Compares response with expected response
3. Publishes verification results

**What happens:**
- Provider tests run automatically (e.g., in CI/CD)
- Verification results (success/failure) are published to the broker
- The broker tracks which provider versions satisfy which consumer contracts

#### Step 4: Deployment Safety Check

Before deploying, use `can-i-deploy` to check if it's safe:
```bash
pact-broker can-i-deploy \
  --pacticipant "users-provider" \
  --to "main" \
  --version "1.0.0" \
  --broker-base-url "http://localhost:9292"
```

**What happens:**
- The broker checks all relevant pacts for the version
- Verifies that all required verifications have passed
- Returns "yes" if safe to deploy, "no" if not

---

## The Pact Workflow

### Complete Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONSUMER WORKFLOW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Write Consumer Test                                        │
│     ↓                                                           │
│  2. Run Consumer Test (generates pact file)                    │
│     ↓                                                           │
│  3. Publish Pact to Broker                                     │
│     ↓                                                           │
│  4. Tag Consumer Version                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                            ↓
                    ┌───────────────┐
                    │  Pact Broker  │
                    │  (Centralized │
                    │   Storage)    │
                    └───────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                    PROVIDER WORKFLOW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Fetch Pacts from Broker                                    │
│     ↓                                                           │
│  2. Run Provider Verification Tests                            │
│     ↓                                                           │
│  3. Publish Verification Results to Broker                      │
│     ↓                                                           │
│  4. Tag Provider Version                                       │
│     ↓                                                           │
│  5. Check can-i-deploy (before deployment)                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Detailed Workflow Steps

#### Phase 1: Consumer Development

1. **Developer writes consumer test**
   ```java
   @Test
   @PactTestFor(pactMethod = "pactWithAllProperties")
   public void runTestPactWithAllProperties(final MockServer server) {
       var template = new RestTemplate();
       var user = template.getForObject(server.getUrl() + "/users/1", User.class);
       assertEquals("1", user.id());
   }
   ```

2. **Run consumer tests**
   ```bash
   ./gradlew test --tests '*Pact*'
   ```
   - Pact framework creates a mock server
   - Test runs against mock server
   - Pact file generated: `build/pacts/users-consumer-users-provider.json`

3. **Publish pact to broker**
   ```bash
   pact-broker publish "./build/pacts" \
     --consumer-app-version "1.0.0" \
     --broker-base-url "http://localhost:9292" \
     --tag "main"
   ```

4. **Tag consumer version** (optional but recommended)
   ```bash
   pact-broker create-version-tag \
     --pacticipant "users-consumer" \
     --version "1.0.0" \
     --tag "main" \
     --broker-base-url "http://localhost:9292"
   ```

#### Phase 2: Provider Verification

1. **Provider fetches pacts** (automatically via `@PactBroker` annotation)
   ```java
   @PactBroker(url = BROKER_URL, providerTags = { "dev" })
   ```

2. **Run provider verification tests**
   ```bash
   PACT_PROVIDER_VERSION="1.0.0" ./gradlew test --tests '*Pact*'
   ```
   - Provider tests fetch pacts from broker
   - For each pact interaction:
     - Set up provider state
     - Make request to provider (or mock)
     - Verify response matches contract
   - Results published to broker automatically

3. **Tag provider version**
   ```bash
   pact-broker create-version-tag \
     --pacticipant "users-provider" \
     --version "1.0.0" \
     --tag "main" \
     --broker-base-url "http://localhost:9292"
   ```

#### Phase 3: Deployment Safety

1. **Check if consumer can deploy**
   ```bash
   pact-broker can-i-deploy \
     --pacticipant "users-consumer" \
     --to "main" \
     --version "1.0.0" \
     --broker-base-url "http://localhost:9292"
   ```
   - Checks: Are all providers that this consumer depends on verified?

2. **Check if provider can deploy**
   ```bash
   pact-broker can-i-deploy \
     --pacticipant "users-provider" \
     --to "main" \
     --version "1.0.0" \
     --broker-base-url "http://localhost:9292"
   ```
   - Checks: Are all consumers' contracts satisfied by this provider version?

---

## Understanding the Components

### Consumer Test Components

#### 1. `@Pact` Annotation
Defines the contract method that generates the pact:
```java
@Pact(provider = "users-provider", consumer = "users-consumer")
public RequestResponsePact pactWithAllProperties(final PactDslWithProvider builder) {
    // Contract definition
}
```

#### 2. `@PactTestFor` Annotation
Links a test method to a specific pact:
```java
@Test
@PactTestFor(pactMethod = "pactWithAllProperties")
public void runTestPactWithAllProperties(final MockServer server) {
    // Test implementation
}
```

#### 3. Mock Server
Pact creates a mock HTTP server that responds according to the contract:
```java
final MockServer server  // Injected by Pact framework
server.getUrl()  // URL of the mock server
```

### Provider Test Components

#### 1. `@Provider` Annotation
Identifies this test class as verifying a specific provider:
```java
@Provider("users-provider")
class UsersPactTest {
    // Provider verification tests
}
```

#### 2. `@PactBroker` Annotation
Configures where to fetch pacts from:
```java
@PactBroker(url = "http://localhost:9292", providerTags = { "dev" })
```

#### 3. `@TestTemplate` and `@ExtendWith`
Enables dynamic test generation for each pact interaction:
```java
@TestTemplate
@ExtendWith(PactVerificationInvocationContextProvider.class)
public void pactVerificationTestTemplate(final PactVerificationContext context) {
    context.verifyInteraction();
}
```

#### 4. `@State` Annotation
Defines provider state handlers:
```java
@State("user-is-leanne")
public void isLeanne() {
    // Set up provider state
}
```

#### 5. WireMock (or Real Provider)
In our example, we use WireMock to simulate the provider:
```java
WireMockServer server = new WireMockServer(wireMockConfig().port(8081));
server.stubFor(get(urlEqualTo("/users/1"))
    .willReturn(aResponse()
        .withBody(BODY)
        .withHeader("content-type","application/json")));
```

**Note**: In production, you'd typically test against the actual provider service, not a mock.

---

## The Pact Broker

### What is the Pact Broker?

The **Pact Broker** is a centralized service that:
- Stores pact files
- Tracks versions of consumers and providers
- Manages verification results
- Provides APIs for querying contract compatibility
- Enables `can-i-deploy` functionality

### Key Features

1. **Version Management**
   - Tracks multiple versions of each consumer and provider
   - Maintains history of contract changes

2. **Verification Tracking**
   - Records which provider versions have verified which consumer contracts
   - Tracks success/failure of verifications

3. **Tagging**
   - Tags represent environments (dev, staging, prod) or branches
   - Enables environment-specific deployment checks

4. **Matrix View**
   - Shows compatibility matrix between consumer and provider versions
   - Visual representation of which versions work together

5. **Web UI**
   - Browser interface for viewing pacts, versions, and verification results
   - Accessible at `http://localhost:9292`

### Broker API Endpoints

- `GET /pacts/provider/{provider}/consumer/{consumer}/latest` - Get latest pact
- `GET /pacts/provider/{provider}/for-verification` - Get pacts for verification
- `POST /pacts/provider/{provider}/consumer/{consumer}/versions/{version}` - Publish pact
- `POST /pacts/provider/{provider}/consumer/{consumer}/pact-version/{version}/verification-results` - Publish verification
- `GET /matrix` - Get compatibility matrix

### Broker Configuration

In our `docker-compose.yml`:
```yaml
pact-broker:
  image: pactfoundation/pact-broker:2.89.1.0
  ports:
    - '9292:9292'
  environment:
    PACT_BROKER_DATABASE_URL: 'postgres://postgres:password@postgres/postgres'
    PACT_BROKER_BASE_URL: 'http://localhost:9292'
```

---

## can-i-deploy: Deployment Safety

### What is can-i-deploy?

`can-i-deploy` is a command that checks whether it's safe to deploy a specific version of a service to a specific environment. It answers the question: **"Can I deploy version X of service Y to environment Z?"**

### How It Works

1. **Identifies relevant pacts**
   - For consumers: Finds all providers this consumer depends on
   - For providers: Finds all consumers that depend on this provider

2. **Checks verification status**
   - Verifies that all required pacts have been successfully verified
   - Ensures verification results are for the correct versions

3. **Returns result**
   - ✅ **"Computer says yes"**: Safe to deploy
   - ❌ **"Computer says no"**: Not safe to deploy (shows what's missing)

### Usage Examples

#### Check if Provider Can Deploy

```bash
pact-broker can-i-deploy \
  --pacticipant "users-provider" \
  --to "main" \
  --version "1.0.0" \
  --broker-base-url "http://localhost:9292"
```

**What it checks:**
- Are all consumers' contracts (tagged with "main") verified by provider version 1.0.0?
- Have all verifications passed?

**Output (Success):**
```
Computer says yes \o/

CONSUMER       | C.VERSION | PROVIDER       | P.VERSION | SUCCESS? | RESULT#
---------------|-----------|----------------|-----------|----------|--------
users-consumer | 1.0.0     | users-provider | 1.0.0     | true     | 1      

All required verification results are published and successful
```

**Output (Failure):**
```
Computer says no ¯_(ツ)_/¯

CONSUMER       | C.VERSION | PROVIDER       | P.VERSION | SUCCESS? | RESULT#
---------------|-----------|----------------|-----------|----------|--------
users-consumer | 1.0.0     | users-provider | 1.0.0     | false    | 1      

The verification for the pact between the latest version of users-consumer 
with tag main (1.0.0) and version 1.0.0 of users-provider failed
```

#### Check if Consumer Can Deploy

```bash
pact-broker can-i-deploy \
  --pacticipant "users-consumer" \
  --to "main" \
  --version "1.0.0" \
  --broker-base-url "http://localhost:9292"
```

**What it checks:**
- Have all providers (tagged with "main") verified this consumer's contracts?
- Are the provider versions compatible with this consumer version?

### Integration with CI/CD

Typically used in deployment pipelines:

```yaml
# Example CI/CD pipeline step
- name: Check if safe to deploy
  run: |
    pact-broker can-i-deploy \
      --pacticipant "users-provider" \
      --to "production" \
      --version "$VERSION" \
      --broker-base-url "$BROKER_URL" || exit 1

- name: Deploy
  run: |
    # Only runs if can-i-deploy passes
    deploy-to-production.sh
```

---

## Failure Scenarios and Detection

### How Pact Detects Breaking Changes

Pact detects breaking changes by comparing:
1. **Request expectations** (method, path, headers, body)
2. **Response expectations** (status, headers, body structure and values)

### Example: Breaking Change Detection

#### Scenario: Provider Changes Response

**Original Contract (Consumer Expects):**
```json
{
  "id": "1",
  "name": "Leanne Graham",
  "username": "leanne",
  "email": "sincere@april.biz"
}
```

**Provider Implementation (Changed):**
```java
// Provider now returns id: "2" instead of "1"
public static final String BODY = """
    {
      "id":"2",  // ❌ BREAKING CHANGE
      "name":"Leanne Graham",
      "username":"leanne",
      "email":"sincere@april.biz"
    }
""";
```

**Test Result:**
```
java.lang.AssertionError: 
Pact between users-consumer (1.0.0) and users-provider - 
Upon Should return all available properties in response body. 
Failures:

1) Verifying a pact between users-consumer and users-provider - 
   Should return all available properties in response body. 
   has a matching body

    1.1) body: $.id Expected '1' (String) but received '2' (String)
```

**can-i-deploy Result:**
```
Computer says no ¯_(ツ)_/¯

CONSUMER       | C.VERSION | PROVIDER       | P.VERSION | SUCCESS? | RESULT#
---------------|-----------|----------------|-----------|----------|--------
users-consumer | 1.0.0     | users-provider | 1.0.0     | false    | 1      

The verification for the pact between the latest version of users-consumer 
with tag main (1.0.0) and version 1.0.0 of users-provider failed
```

### Common Breaking Changes

1. **Changed Field Values**
   - Provider returns different data than expected
   - **Detection**: Field value mismatch

2. **Missing Fields**
   - Provider removes a field that consumer expects
   - **Detection**: Missing field in response

3. **Changed Field Types**
   - Provider changes field type (string → number)
   - **Detection**: Type mismatch

4. **Changed HTTP Status**
   - Provider returns different status code
   - **Detection**: Status code mismatch

5. **Changed Headers**
   - Provider changes or removes expected headers
   - **Detection**: Header mismatch

6. **Changed Request Format**
   - Consumer sends request in format provider no longer accepts
   - **Detection**: Request validation failure

### Fixing Breaking Changes

1. **Identify the breaking change** from test output
2. **Update provider** to match contract (or update contract if change is intentional)
3. **Re-run provider tests** to verify fix
4. **Re-check can-i-deploy** to confirm deployment safety

**Example Fix:**
```java
// Revert to match contract
public static final String BODY = """
    {
      "id":"1",  // ✅ Fixed
      "name":"Leanne Graham",
      "username":"leanne",
      "email":"sincere@april.biz"
    }
""";
```

---

## Best Practices

### 1. Consumer Practices

#### Define Minimal Contracts
- Only include fields the consumer actually uses
- Don't include fields "just in case"
- This gives providers flexibility to add fields without breaking contracts

#### Use Provider States
- Use provider states to test different scenarios
- Example: `"user-exists"`, `"user-not-found"`, `"user-with-permissions"`

#### Version Your Contracts
- Use semantic versioning for consumer versions
- Tag versions appropriately (dev, staging, prod)

#### Test Real Scenarios
- Test actual use cases, not theoretical ones
- Ensure contracts reflect real API usage

### 2. Provider Practices

#### Verify Frequently
- Run provider verification tests in CI/CD
- Verify against all consumer versions, not just latest

#### Handle Multiple Consumer Versions
- Support multiple consumer contract versions during transitions
- Use version tags to manage compatibility

#### Set Up Provider States Properly
- Implement state handlers that actually set up the required state
- In production, this might involve database setup, API mocking, etc.

#### Monitor Verification Results
- Track verification success rates
- Alert on verification failures

### 3. Broker Practices

#### Use Tags for Environments
- Tag versions with environment names (dev, staging, prod)
- Use tags for branches (main, feature-x)

#### Regular Cleanup
- Archive old versions periodically
- Keep broker database size manageable

#### Secure the Broker
- Use authentication in production
- Restrict access appropriately

### 4. CI/CD Integration

#### Consumer Pipeline
```yaml
1. Run consumer tests
2. Generate pact files
3. Publish pacts to broker
4. Tag consumer version
5. Trigger provider verification (via webhook or polling)
```

#### Provider Pipeline
```yaml
1. Fetch pacts from broker
2. Run provider verification tests
3. Publish verification results
4. Tag provider version
5. Check can-i-deploy
6. Deploy if safe
```

#### Deployment Gate
```yaml
- Always run can-i-deploy before deployment
- Block deployment if can-i-deploy fails
- Make can-i-deploy a required check in CI/CD
```

### 5. Team Collaboration

#### Communication
- Notify teams when contracts change
- Discuss breaking changes before implementing
- Use pull requests for contract reviews

#### Documentation
- Document provider states and their meanings
- Explain contract evolution over time
- Maintain changelog of contract versions

---

## Real-World Example Walkthrough

Let's walk through the complete flow using our example project.

### Setup

1. **Start Pact Broker**
   ```bash
   docker-compose up
   ```
   - Starts PostgreSQL database
   - Starts Pact Broker on port 9292

2. **Set Environment Variables**
   ```bash
   export BROKER_URL="http://localhost:9292"
   export CONSUMER_VERSION="1.0.0"
   export PROVIDER_VERSION="1.0.0"
   export TAG="main"
   ```

### Step 1: Consumer Defines Contract

**File**: `consumer/src/test/java/com/hypeflame/UsersPactTest.java`

The consumer test defines:
- **Request**: GET /users/1
- **Response**: Status 200 with user data
- **Provider State**: "user-is-leanne"

**Run Consumer Tests:**
```bash
./consumer/gradlew -p ./consumer test --tests '*Pact*'
```

**Result:**
- ✅ Tests pass
- ✅ Pact file generated: `consumer/build/pacts/users-consumer-users-provider.json`

### Step 2: Publish Pact to Broker

```bash
docker run --rm --network pact-broker-tests-main_default \
  -v $(pwd)/consumer/build/pacts:/pacts \
  pactfoundation/pact-cli:latest broker publish /pacts \
  --consumer-app-version "$CONSUMER_VERSION" \
  --broker-base-url "http://pact-broker:9292" \
  --tag "$TAG"
```

**Result:**
- ✅ Pact published to broker
- ✅ Consumer version 1.0.0 tagged with "main"
- ✅ Viewable at: http://localhost:9292

### Step 3: Provider Verifies Contract

**File**: `provider/src/test/java/com/hypeflame/UsersPactTest.java`

The provider test:
- Fetches pacts from broker (via `@PactBroker`)
- Sets up WireMock to simulate provider API
- Verifies response matches contract

**Run Provider Tests:**
```bash
PACT_PROVIDER_VERSION="$PROVIDER_VERSION" \
  ./provider/gradlew -p ./provider test --tests '*Pact*'
```

**Result:**
- ✅ Tests pass
- ✅ Verification results published to broker

### Step 4: Tag Provider Version

```bash
docker run --rm --network pact-broker-tests-main_default \
  pactfoundation/pact-cli:latest broker create-version-tag \
  --pacticipant "users-provider" \
  --version "$PROVIDER_VERSION" \
  --tag "$TAG" \
  --broker-base-url "http://pact-broker:9292"
```

### Step 5: Check Deployment Safety

**Check Provider:**
```bash
docker run --rm --network pact-broker-tests-main_default \
  pactfoundation/pact-cli:latest broker can-i-deploy \
  --pacticipant "users-provider" \
  --to "$TAG" \
  --version "$PROVIDER_VERSION" \
  --broker-base-url "http://pact-broker:9292"
```

**Result:**
```
Computer says yes \o/
✅ Safe to deploy
```

### Step 6: Simulate Breaking Change

**Change Provider Response:**
```java
// Change id from "1" to "2"
public static final String BODY = """
    {
      "id":"2",  // ❌ Breaking change
      ...
    }
""";
```

**Run Provider Tests:**
```bash
PACT_PROVIDER_VERSION="$PROVIDER_VERSION" \
  ./provider/gradlew -p ./provider test --tests '*Pact*'
```

**Result:**
```
❌ Test fails:
Expected '1' (String) but received '2' (String)
```

**Check can-i-deploy:**
```bash
pact-broker can-i-deploy ...
```

**Result:**
```
Computer says no ¯_(ツ)_/¯
❌ Not safe to deploy
```

### Step 7: Fix and Verify

**Revert Change:**
```java
public static final String BODY = """
    {
      "id":"1",  // ✅ Fixed
      ...
    }
""";
```

**Re-run Tests:**
```bash
./provider/gradlew -p ./provider clean test --tests '*Pact*'
```

**Result:**
```
✅ Tests pass
✅ can-i-deploy says "yes"
✅ Safe to deploy
```

---

## Conclusion

Pact contract testing provides a powerful way to ensure API compatibility between services in a microservices architecture. By:

1. **Defining contracts in code** (not documents)
2. **Testing independently** (no need for all services running)
3. **Catching breaking changes early** (before deployment)
4. **Managing versions centrally** (via Pact Broker)
5. **Enabling safe deployments** (via can-i-deploy)

You can build more reliable, maintainable microservices with confidence that services will work together correctly.

### Key Takeaways

- ✅ **Consumer-driven**: Consumers define what they need
- ✅ **Fast feedback**: Tests run quickly without complex setup
- ✅ **Early detection**: Breaking changes caught before deployment
- ✅ **Version management**: Track compatibility over time
- ✅ **Deployment safety**: can-i-deploy prevents breaking deployments

### Next Steps

1. Integrate Pact into your CI/CD pipelines
2. Set up the Pact Broker for your team
3. Start with one consumer-provider pair
4. Expand to more services gradually
5. Use can-i-deploy as a deployment gate

---

## Additional Resources

- [Pact Documentation](https://docs.pact.io/)
- [Pact Broker Documentation](https://docs.pact.io/pact_broker)
- [Pact-JVM Documentation](https://github.com/DiUS/pact-jvm)
- [Consumer-Driven Contracts](https://martinfowler.com/articles/consumerDrivenContracts.html)

---

*This guide was created based on the pact-broker-tests project. For hands-on practice, follow the steps in the README.md file.*

