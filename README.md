# 🚗 SafeDriveEt (SafeDrive Ethiopia)

SafeDriveEt is a state-of-the-art, event-driven microservices ride-hailing and taxi platform tailored for the Ethiopian market. It is engineered for low-latency driver matching, resilient asynchronous payments, and real-time WebSocket communication.

Built using **Go (Golang)**, **Next.js**, **RabbitMQ**, **Docker**, and orchestration on **Kubernetes (Tilt & GKE)**.

---

## 🏗️ System Architecture & Event Flow

SafeDriveEt uses an **asynchronous event-driven choreography** pattern powered by RabbitMQ. This ensures high availability, decouples heavy computations (like routing and payment processing) from the main client flow, and ensures fault-tolerant operations.

### 1. Trip Booking & Lifecycle Flow
Below is the sequence diagram illustrating how a user requests a ride, gets matched with a driver, and pays securely:

```mermaid
sequenceDiagram
  participant User as Client App (Next.js)
  participant APIGateway as API Gateway (Go)
  participant TripService as Trip Service (Go)
  participant DriverService as Driver Service (Go)
  participant Driver as Driver App / Client
  participant PaymentService as Payment Service (Go)
  participant Stripe as Stripe API
  participant OSRM as OSRM Routing API

  User ->> APIGateway: HTTP GET /trips/preview (Route preview)
  APIGateway ->> TripService: gRPC: PreviewTrip
  TripService ->> OSRM: HTTP: Calculate Route coordinates
  OSRM ->> TripService: Route & ETAs
  TripService ->> APIGateway: Return route & pricing
  APIGateway ->> User: Render map preview & fare estimates

  User ->> APIGateway: HTTP POST /trips (Request Trip)
  APIGateway ->> TripService: gRPC: CreateTrip
  Note over TripService, DriverService: Trip Event Exchange (RabbitMQ)
  TripService -->>+ DriverService: Event: trip.event.created
  DriverService ->> Driver: WebSocket: driver.cmd.trip_request
  
  Driver ->> APIGateway: WebSocket: driver.cmd.trip_accept
  APIGateway -->> TripService: AMQP: driver.cmd.trip_accept
  
  Note over TripService: Process match & set status: Assigned
  TripService -->> PaymentService: Event: trip.event.driver_assigned
  
  PaymentService ->> Stripe: Create Checkout Session
  Stripe -->> PaymentService: Session Created
  PaymentService -->> APIGateway: Event: payment.event.session_created
  APIGateway ->> User: WebSocket: Show payment screen
  
  User ->> Stripe: Complete payment checkout
  Stripe ->> APIGateway: Webhook callback: payment.event.success
  APIGateway -->>+ TripService: Event: payment.event.success
  Note right of TripService: Trip goes active, driver pick-up begins
```

### 2. RabbitMQ Message Distribution
The system uses dedicated AMQP Exchanges and routing keys to distribute events to respective consumer queues:

```mermaid
graph TD
    subgraph Exchanges[AMQP Exchanges]
        TE[Trip Exchange]
        PE[Payment Exchange]
    end

    subgraph Queues[RabbitMQ Queues]
        Q1[find_available_drivers]
        Q2[notify_new_trip]
        Q3[notify_driver_assignment]
        Q4[notify_driver_no_drivers_found]
        Q5[driver_cmd_trip_request]
        Q6[driver_trip_response]
        Q7[create_payment_session]
        Q8[notify_payment_status]
        Q9[payment_success_trip_update]
    end

    subgraph Events[Message Routing Keys]
        E1[trip.event.created]
        E2[trip.event.driver_assigned]
        E3[trip.event.no_drivers_found]
        E4[trip.event.cancelled]
        E5[driver.cmd.trip_request]
        E6[driver.cmd.trip_accept]
        E7[driver.cmd.trip_decline]
        E8[payment.event.session_created]
        E9[payment.event.success]
        E10[payment.event.failed]
    end

    subgraph Services[Microservices]
        TS[Trip Service]
        DS[Driver Service]
        AG[API Gateway]
        PS[Payment Service]
    end

    %% Event Routing to Queues
    E1 --> Q1
    E1 --> Q2
    E2 --> Q3
    E2 --> Q7
    E3 --> Q4
    E5 --> Q5
    E6 --> Q6
    E7 --> Q6

    E8 --> Q8
    E9 --> Q9
    E10 --> Q9

    %% Services Publishing Events
    TS -.->|Publish| TE
    DS -.->|Publish| TE
    PS -.->|Publish| PE

    %% Queue Consumers
    Q1 --> DS
    Q2 --> AG
    Q3 --> AG
    Q4 --> AG
    Q5 --> DS
    Q6 --> TS
    Q7 --> PS
    Q8 --> AG
    Q9 --> TS

    style Exchanges fill:#e6b3ff,stroke:#333,stroke-width:2px
    style Services fill:#80b3ff,stroke:#333,stroke-width:2px
    style Events fill:#ffb366,stroke:#333,stroke-width:2px
    style Queues fill:#85e085,stroke:#333,stroke-width:2px
```

---

## 🗂️ Project Structure

- **`/services`**: Microservice implementations.
  - **`api-gateway/`**: Built in Go; acts as the primary proxy. Translates external HTTP/WebSocket clients' requests into internal gRPC calls and AMQP messages.
  - **`trip-service/`**: Built in Go; encapsulates trip logic, route parsing, and pricing schemes.
  - **`driver-service/`**: Built in Go; maintains driver status (availability, geohash locations) and executes the matching logic.
  - **`payment-service/`**: Built in Go; coordinates Stripe checkout, Chapa integration, and tracks payment events.
- **`/web`**: Web app client dashboard powered by Next.js, React, TailwindCSS, and shadcn/ui.
- **`/shared`**: Standard Go modules reused across all microservices (DB wrappers, env parsers, contract schemas, utilities).
- **`/infra`**: Kubernetes templates and Dockerfiles.
  - `infra/development/`: Manifests, Tiltfile configuration, and Dockerfiles optimized for fast hot-reload during local dev.
  - `infra/production/`: Hardened manifests and multi-stage Dockerfiles prepared for GKE deployment.

---

## 🚀 Local Development Setup

To run SafeDriveEt locally, you will need a Kubernetes cluster and Tilt, which manages live-reloading of microservices within your cluster.


### Step-by-Step Installation

#### 1. Setup local Kubernetes Cluster (e.g., Minikube)
Start minikube and enable the ingress controller:
```bash
minikube start --driver=docker
minikube addons enable ingress
```

#### 2. Install Go Dependencies
Ensure the shared modules and dependencies are clean:
```bash
go mod tidy
```

#### 3. Run with Tilt
Launch the Tilt dashboard to build, deploy, and monitor the microservices automatically:
```bash
tilt up
```
Once Tilt is running, press **`Space`** or open [http://localhost:10350](http://localhost:10350) in your browser to access the Tilt UI. It compiles the Go binaries, builds Docker images, deploys K8s resources, and streams live logs.

- **API Gateway** is forwarded on `http://localhost:8081`
- **Next.js Web Frontend** is accessible at `http://localhost:3000`

---

## 🔍 Monitoring & Logging

### View Cluster Pods
```bash
kubectl get pods -A
```

### Access Kubernetes Dashboard
```bash
minikube dashboard
```

### Distributed Tracing (Jaeger)
If Jaeger tracing is enabled in the development namespace, you can access the tracing UI to analyze network flows and latency:
```bash
kubectl port-forward deployment/jaeger 16686:16686
# Open http://localhost:16686
```

---

