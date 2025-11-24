# Pact Contract Testing Setup (TypeScript Consumer + Java/Gradle Provider)

This document contains ready-to-use examples for:

- A **TypeScript consumer** (e.g. `BookingFrontend`) using Pact JS
- A **Java/Gradle provider** (e.g. `BookingService`) using the Pact JVM Gradle plugin
- **GitHub Actions** workflows to:
  - run the consumer Pact tests and publish contracts
  - run provider verification against a deployed dev backend

You can copy sections directly into your repos.

---

## 1. TypeScript Consumer (BookingFrontend)

### 1.1. Example HTTP client

`src/bookingApiClient.ts`:

```ts
import axios, { AxiosResponse } from "axios";

export interface Booking {
  id: string;
  customerName: string;
  startDate: string;
}

export class BookingApiClient {
  constructor(private readonly baseUrl: string) {}

  async getBooking(id: string): Promise<AxiosResponse<Booking>> {
    return axios.request<Booking>({
      baseURL: this.baseUrl,
      url: `/bookings/${id}`,
      method: "GET",
      headers: { Accept: "application/json" },
    });
  }
}
```

---

### 1.2. Pact consumer test (TypeScript + Jest)

`test/bookingApiClient.pact.test.ts`:

```ts
import path from "path";
import { PactV3, MatchersV3 } from "@pact-foundation/pact";
import { BookingApiClient } from "../src/bookingApiClient";

const provider = new PactV3({
  dir: path.resolve(process.cwd(), "pacts"),
  consumer: "BookingFrontend",
  provider: "BookingService",
});

describe("BookingFrontend -> BookingService contract", () => {
  it("GET /bookings/:id returns a booking", async () => {
    const bookingId = "123";
    const bookingExample = {
      id: bookingId,
      customerName: "Alice Smith",
      startDate: "2025-01-01",
    };

    const EXPECTED_BODY = MatchersV3.like(bookingExample);

    provider
      .given("a booking with id 123 exists")
      .uponReceiving("a request for booking 123")
      .withRequest({
        method: "GET",
        path: `/bookings/${bookingId}`,
        headers: {
          Accept: "application/json",
        },
      })
      .willRespondWith({
        status: 200,
        headers: {
          "Content-Type": "application/json; charset=utf-8",
        },
        body: EXPECTED_BODY,
      });

    return provider.executeTest(async (mockServer) => {
      const client = new BookingApiClient(mockServer.url);
      const response = await client.getBooking(bookingId);

      expect(response.status).toBe(200);
      expect(response.data.id).toBe(bookingId);
      expect(response.data.customerName).toBe("Alice Smith");
    });
  });
});
```

This generates a pact file:

- `pacts/BookingFrontend-BookingService.json`

---

### 1.3. `package.json` scripts

In `packages/booking-frontend/package.json`:

```jsonc
{
  "scripts": {
    "test:pact": "jest --runInBand test/bookingApiClient.pact.test.ts"
  },
  "jest": {
    "preset": "ts-jest",
    "testEnvironment": "node"
  }
}
```

You’ll also need the dependencies (at least):

```bash
npm install --save-dev @pact-foundation/pact jest ts-jest @types/jest axios @types/axios
```

---

## 2. GitHub Actions – Consumer Pact Tests + Publish

`.github/workflows/booking-frontend-pact.yml`:

```yaml
name: BookingFrontend Pact Consumer

on:
  push:
    paths:
      - "packages/booking-frontend/**"
  workflow_dispatch: {}

jobs:
  pact-consumer:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: packages/booking-frontend

    env:
      PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_BASE_URL }}
      PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
      CONSUMER_APP_VERSION: ${{ github.sha }}
      CONSUMER_BRANCH: ${{ github.ref_name }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install dependencies
        run: npm ci

      - name: Run Pact tests
        run: npm run test:pact

      - name: Publish pacts to broker
        run: |
          npx pact-broker publish ./pacts             --consumer-app-version="$CONSUMER_APP_VERSION"             --branch="$CONSUMER_BRANCH"             --broker-base-url="$PACT_BROKER_BASE_URL"             --broker-token="$PACT_BROKER_TOKEN"
```

Make sure these secrets exist in your repo:

- `PACT_BROKER_BASE_URL`
- `PACT_BROKER_TOKEN`

---

## 3. Java + Gradle Provider (BookingService)

### 3.1. Gradle build file

`booking-service/build.gradle` (Groovy DSL):

```groovy
plugins {
  id 'java'
  id 'au.com.dius.pact' version '4.6.18' // or latest 4.6.x
}

repositories {
  mavenCentral()
}

dependencies {
  testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
  testImplementation 'org.springframework.boot:spring-boot-starter-test' // if you use Spring
}

test {
  useJUnitPlatform()
}

pact {
  broker {
    pactBrokerUrl = System.getenv("PACT_BROKER_BASE_URL")
    pactBrokerToken = System.getenv("PACT_BROKER_TOKEN")
  }

  serviceProviders {
    BookingService {
      protocol = 'https'
      host = System.getenv("BOOKING_SERVICE_HOST") ?: "booking-service-dev.yourcompany.com"
      port = (System.getenv("BOOKING_SERVICE_PORT") ?: "443") as Integer
      path = '/'

      providerVersion = { System.getenv("PROVIDER_APP_VERSION") ?: project.version }

      fromPactBroker {
        withSelectors {
          mainBranch()
          // Other options: deployedOrReleased(), environment('dev'), etc.
        }
      }
    }
  }
}
```

This sets up a `pactVerify` Gradle task that:

- pulls pacts for `BookingService` from the Broker
- hits your dev BookingService at the configured host/port
- ties the results to `PROVIDER_APP_VERSION`

To publish verification results to the Broker, run with:

```bash
./gradlew pactVerify -Ppact.verifier.publishResults=true
```

---

## 4. GitHub Actions – Provider Verification (Gradle)

`.github/workflows/booking-service-pact-provider.yml`:

```yaml
name: BookingService Pact Provider

on:
  push:
    paths:
      - "booking-service/**"
  workflow_dispatch: {}

jobs:
  pact-provider:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: booking-service

    env:
      PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_BASE_URL }}
      PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
      PROVIDER_APP_VERSION: ${{ github.sha }}
      BOOKING_SERVICE_HOST: "booking-service-dev.yourcompany.com"
      BOOKING_SERVICE_PORT: "443"

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: "temurin"
          java-version: "21"

      - name: Build & unit tests
        run: ./gradlew clean test --no-daemon

      - name: Verify pacts from Broker against BookingService (dev)
        run: |
          ./gradlew pactVerify             -Ppact.verifier.publishResults=true             --no-daemon
```

This job will:

1. Build and test your Java service.
2. Run Pact provider verification against your **deployed dev backend**.
3. Publish verification results back to the Pact Broker.

---

## 5. Next Steps (Optional)

Once this is working, you can add:

- `pact-broker can-i-deploy` in your promotion pipelines (preprod/prod)
- `pact-broker record-deployment` to track which versions are deployed where
- `pact-broker record-release` for tracking released mobile versions

But those are **not required** for getting the basic consumer/provider contract testing loop working.
