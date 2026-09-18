# Auction Platform

A full-stack Spring Boot and React auction application demonstrating layered architecture, REST APIs, real-time bidding, and payment processing.

**Note:** This is a forked version of a group project. This README documents my specific backend contributions to the payment and auction systems.

---

## My Contributions

### **Auction Winner Determination**

Implemented a winner-determination algorithm in `AuctionService`, iterating through several design passes:

- **`winningRule(AuctionResultDTO details)`** — initial skeleton with winner retrieval logic
- **`getWinner(Long auctionID)`** — full `@Transactional` implementation: validated auction state (checks if the auction has ended), iterated all bids to find the highest amount, matched it to the bidder, and persisted the result via `auctionRepository.save()`
- Fixed a bug where auction status was read from a stale DTO instead of the live repository state, and added `@Transactional` handling for consistency

After building and testing this end-to-end, I identified that it duplicated an existing `getWinningBidder()` accessor already in the codebase and removed the redundant method rather than ship duplicate logic.

**What's live today:** a `GET /{auctionId}/winner` endpoint in `AuctionController`, which retrieves the highest bidder from the auction and returns their details via a `WinnerDTO` (built alongside the DTOs below) — this is what the frontend calls to display winner info on the payment page.

---

### **Payment Processing System**

Built complete payment service layer from scratch, extracting logic into dedicated `PaymentService` class:

#### **PaymentService (`placePayment` Method)**

Implemented full payment placement workflow:

- **Auction Validation:** Checks if auction has ended before allowing payment
- **Payee Verification:** Retrieves and validates payee exists in system
- **Payment Validation:**
  - Credit card number must be exactly 16 digits
  - Expiry date must not be before current time
  - Security code must be exactly 3 digits
- **Payment Object Creation:** Uses builder pattern with all required fields:
  - `paymentID` — auto-generated
  - `auction` — reference to auction entity
  - `payee` — winning bidder
  - `paymentDate` — timestamp (OffsetDateTime.now())
  - `expectedDeliveryDate` — 7 days after payment
- **Persistence:** Saves payment to database via `paymentRepository`
- **Response Building:** Constructs `PaymentResponseDTO` with delivery details
- **Error Handling:** Exception handling for auction not found, payee not found, payment failures

**Database Retrieval Optimization:**
- Refactored to use `findDetailedById()` custom query
- Eagerly loads all related entities (payee with address, bids, auction, item) to prevent N+1 queries

#### **PaymentService (`createReceipt` Method)**

Implemented receipt generation with complex data aggregation:

- **Payment Verification:** Retrieves stored payment from repository before creating receipt
- **Bid Analysis:** Iterates through winning user's bids to find largest amount bid
- **Price Calculation:** Computes total price as base price (highest bid amount) + shipping cost
- **Receipt DTO Assembly:** Builds `ReceiptResponseDTO` with:
  - Payee details (firstName, lastName, full address)
  - Auction/item information (itemID, itemName)
  - Pricing (totalPaid, shippingDate)
  - Delivery information (expectedDeliveryDate)
- **Exception Handling:** Returns error receipt if payment not found

---

### **Data Transfer Objects (DTOs)**

Designed and implemented 5 DTOs for clean API contracts and data transfer:

**PaymentRequestDTO** (15 fields)
- Payment details: paymentID, auctionID
- Cardholder info: firstName, lastName, streetName, streetNumber, city, country, postalCode
- Card details: cardNumber (String), nameOnCard, expiryDate (OffsetDateTime), securityCode (String), isExpedited (boolean)
- User reference: user (User object)
- Uses Lombok (@Data, @Builder, @NoArgsConstructor, @AllArgsConstructor)

**PaymentResponseDTO** (4 fields)
- Response data: paymentID, firstName, lastName, deliveryDate
- Message confirmation ("Payment placed successfully.")
- Uses Lombok annotations

**PaymentDTO** (3 fields)
- Simplified payment reference: paymentID, auction, payee
- Used for payment object handling

**PaymentDetailDTO** (5 fields)
- Complete payment details: paymentID, auction, payee, paymentDate, expectedDeliveryDate
- Retrieved from database for detailed payment information

**ReceiptResponseDTO** (11 fields)
- Receipt details with address: firstName, lastName, streetName, streetNumber, city, country, postalCode
- Item info: itemID, totalPaid (BigDecimal), shippingDate (OffsetDateTime)
- Message confirmation ("Receipt generated.")

**WinnerDTO**
- Surfaces winning-bidder identity and address fields to the frontend for display on the payment page

---

### **Payment Entity & Persistence**

Designed `Payment` JPA entity with proper relationships:

- **Fields:** paymentID (auto-generated), auction (ManyToOne), payee (User), paymentDate, expectedDeliveryDate, isExpedited
- **Relationships:** Properly configured @JoinColumn for auction reference
- **Annotations:** @Entity, @Table("payments"), @Lombok utilities
- **Exclusions:** Used @ToString.Exclude and @EqualsAndHashCode.Exclude for relationship handling to prevent circular references

**PaymentRepository**
- Created Spring Data JPA repository interface extending `JpaRepository<Payment, Long>`
- Implemented custom `findDetailedById()` query with FETCH joins:
  ```sql
  SELECT p FROM Payment p
  JOIN FETCH p.payee u
  LEFT JOIN FETCH u.address
  LEFT JOIN FETCH u.bids b
  JOIN FETCH p.auction a
  JOIN FETCH a.item i
  WHERE p.paymentID = :id
  ```
- Prevents N+1 query problems by eagerly loading all related entities in single query

---

### **REST API Endpoints**

Implemented `PaymentController` (base path `/payment`) with three endpoints:

**GET `/payment/{paymentId}`**
- Retrieves payment details by ID
- Calls `paymentService.getPaymentDetails(paymentId)`
- Returns `ResponseEntity<PaymentDetailDTO>`

**POST `/payment/place`**
- Accepts `PaymentRequestDTO` in request body
- Calls `paymentService.placePayment(request)`
- Returns `ResponseEntity<PaymentResponseDTO>` with payment confirmation

**GET `/payment/receipt/{paymentId}`**
- Accepts the payment ID as a path variable
- Calls `paymentService.createReceipt(paymentId)`
- Returns `ResponseEntity<ReceiptResponseDTO>` with receipt details
- (This endpoint went through a couple of iterations — it started as `POST /auction/receipt` taking a full `Payment` object, and was refactored down to a `GET` taking just the ID, to match how the frontend actually needed to call it)

---

### **Refactoring & Architecture Improvements**

**Service Layer Separation:**
- Extracted payment logic from `AuctionService` into dedicated `PaymentService` class
- Removed `placePayment()` and `createReceipt()` methods from auction service
- Improved separation of concerns and testability

**Datetime Handling:**
- Refactored `PaymentRequestDTO` and `PaymentResponseDTO` to use `OffsetDateTime` instead of `Date`
- Ensures consistency with project-wide datetime standards

**Data Type Fixes:**
- Changed `cardNumber` and `securityCode` from `Long` to `String` in `PaymentRequestDTO` for proper validation
- Changed `totalPaid` from `Float` to `BigDecimal` in `ReceiptResponseDTO` for precise monetary calculations

**Import Cleanup:**
- Removed unused imports across payment-related classes
- Kept only necessary Spring, JPA, Lombok, and Java imports

---

### **Frontend Implementation (React/TypeScript)**

Built the client-side bidding, payment, and receipt flow from scratch, wiring each screen to the backend endpoints above.

**`BidForm`**
- Controlled form for entering a bid amount, with a dedicated header and a clear "Submit Bid" call-to-action
- Validates the entered amount against the current highest bid before allowing submission, with error states surfaced for insufficient or invalid amounts
- On submit, passes the bid amount and auction ID to `placeBid` and routes the user to the auction detail page

**`AuctionDetailPage`**
- Live countdown timer computed from the current time and the auction's end time, updating the displayed time remaining and switching to an "auction ended" state once time expires
- Fetches and displays the winning bidder's info (via `WinnerDTO`) once an auction has ended
- Redirect handling so a bidder who did not win cannot land on the payment page for that auction

**`PaymentForm` / `PaymentPage`**
- Full payment form covering card number, name on card, expiry date, security code, and an expedited-shipping checkbox
- Client-side validation mirrors the backend rules: digit-only input masking and length limits on the card number and security code fields, with per-field error messages before submission is allowed
- Expedited-shipping selection is factored into the delivery date and total price shown to the user
- Submits the assembled payment payload to `placePayment` and displays a confirmation once the backend responds

**`ReceiptPage`**
- Retrieves the generated receipt by payment ID and displays payee details, shipping details, and total price paid
- Handles the case where the receipt hasn't loaded yet before rendering

**API layer**
- Authored `bidAPI.ts` (`placeBid`) and `paymentAPI.ts` (`placePayment`) from scratch to connect the forms above to their respective backend endpoints
- Added a shared `authHeader()` helper and applied it to existing auction/search API calls (`auctionApi.ts`) so authenticated requests correctly attach the bearer token

**Styling**
- Styled the bid, payment, and receipt views (`auctionStyles.css`) to match the site's overall visual design

---

## Tech Stack

**Backend:**
- Java 17, Spring Boot 3.x
- Spring Data JPA/Hibernate with custom queries
- Spring MVC REST
- Lombok for boilerplate reduction
- Jakarta Persistence annotations

**Database:**
- PostgreSQL with JPA entity relationships
- Custom @Query for complex JOIN FETCH scenarios

**Frontend:**
- React, TypeScript, Vite

---

## What I Learned

This project reinforced several key backend and full-stack principles:

1. **Layered Architecture** — Clean separation between controllers, services, repositories, and DTOs
2. **Database Optimization** — Using FETCH joins to avoid N+1 query problems in complex scenarios
3. **Business Logic Encapsulation** — Implementing core algorithms (winner determination, payment validation) with proper exception handling
4. **Data Integrity** — Using `@Transactional` and proper entity relationships to maintain consistency
5. **API Design** — Building RESTful endpoints with appropriate DTOs and error handling
6. **Full-Stack Consistency** — Mirroring backend validation rules (card number length, security code format) on the client for immediate user feedback, while keeping the backend as the source of truth

---

## Original Project

Built as part of EECS 4413 (Building E-Commerce Systems) at York University.
Original repository: [jhaniff/EECS4413-Auction-Site](https://github.com/jhaniff/EECS4413-Auction-Site)

---

## Original Project

Built as part of EECS 4413 (Building E-Commerce Systems) at York University.
Original repository: [jhaniff/EECS4413-Auction-Site](https://github.com/jhaniff/EECS4413-Auction-Site)

---
# Auction Platform (EECS-4413)

## 🧾 Project Overview
This project is a **real-time online auction platform** developed as part of the EECS 4413 course.  
It allows users to register, browse items, place bids, and make mock payments through a secure and modular web application.  

The main goal of the system is to demonstrate a **clean, layered architecture** using **Spring Boot** and modern web technologies.  
It emphasizes maintainability, scalability, and adherence to software engineering best practices such as separation of concerns, modularization, and design pattern use.

---

## 🧱 Tech Stack

### **Backend**
- **Java 17** — primary programming language  
- **Spring Boot 3.x** — main application framework  
- **Spring Web / MVC** — handles REST APIs and routing  
- **Spring Data JPA (Hibernate)** — object-relational mapping (ORM) for database access  
- **Spring Security (JWT-based)** — authentication and authorization  
- **PostgreSQL** — main relational database
- **Flyway** — For database create and seed data
- **Jakarta Validation** — input validation across controllers and DTOs  

### **Frontend (for D3)**
- **React + TypeScript** — user interface framework  
- **Vite** — fast development build tool  
- **Tailwind CSS** — component styling  
- **React Query** — state management and data synchronization  
- **SSE (Server-Sent Events)** — real-time updates for live bidding  

### **DevOps / Tooling**
- **Maven** — build automation and dependency management  
- **Docker & Docker Compose** — containerized services and local environment setup  
- **JUnit 5 / Mockito** — unit and integration testing  

---

## 🧩 Architecture Overview

The system follows a **multi-module, microservice-ready architecture**, designed around clear separation of responsibilities:


---

## ⚙️ Architectural Layers (per Service)

Each service (user, catalogue, auction, payment) follows the **Spring layered architecture pattern**:

1. **Controller Layer** – handles incoming REST requests and responses.  
   - Uses DTOs (Data Transfer Objects) to communicate with clients.  
   - Validates input using Jakarta Bean Validation.

2. **Service Layer** – contains **business logic** and orchestrates repository operations.  
   - Enforces rules like “bids must be strictly increasing” and “only logged-in users can bid”.

3. **Repository Layer** – uses **Spring Data JPA** for database persistence.  
   - Encapsulates all SQL logic and entity management.

4. **Entity / Model Layer** – maps database tables to Java objects (JPA entities).  
   - Each entity includes validation constraints and relationships (e.g., OneToMany).

---

## 🧠 Design Patterns Used

- **Model–View–Controller (MVC):**  
  Used within each Spring Boot service to separate web requests (Controller), business logic (Service), and persistence (Repository).

- **Repository Pattern:**  
  Encapsulates data access logic, promoting abstraction and easier testing.

- **DTO (Data Transfer Object) Pattern:**  
  Prevents entity exposure in APIs and defines consistent data contracts between layers and services.

- **Dependency Injection (DI):**  
  Managed automatically by Spring for modular, testable code.

- **Observer Pattern (via SSE):**  
  Used in the Auction Service to notify clients in real time when new bids are placed.

- **Singleton Pattern:**  
  Applied implicitly through Spring-managed beans (e.g., services and repositories).

---

## 🏗️ Overall Design Philosophy

- **Separation of Concerns:** Each microservice handles a single bounded context.  
- **Scalability:** Independent services allow horizontal scaling if required.  
- **Resilience:** API and service layers remain loosely coupled.  
- **Testability:** Use of MockMvc, JUnit, Mockito, and clean layering enables automated testing.  
- **Extensibility:** Ready for D3 integration with React UI and optional real-time WebSocket communication.  

---

## 🧾 Summary

| Layer / Module       | Responsibility                                  | Tech Used                  |
|----------------------|--------------------------------------------------|-----------------------------|
| **User Service**     | Auth, signup, login, profiles, JWT               | Spring Boot, JPA, BCrypt    |
| **Catalogue Service**| Item listings, search, metadata                 | Spring Boot, JPA, PostgreSQL|
| **Auction Service**  | Bids, timers, SSE stream                        | Spring Boot, JPA            |
| **Payment Service**  | Mock payments, receipt generation               | Spring Boot, Validation     |
| **Frontend (D3)**    | User UI for browsing and bidding                | React, TypeScript, Tailwind |

---

## 🧭 In Summary
The Auction Platform demonstrates how a distributed, layered, and pattern-driven architecture can be implemented using **Spring Boot** while maintaining clarity, modularity, and extendability. It provides a clean foundation for both academic demonstration and scalable production systems.

---

## 🧪 Local Setup & End-to-End Smoke Test

1. **Install prerequisites**
  - Java 17 (LTS). Verify with `java -version` and ensure the runtime reports 17.x.
  - PostgreSQL 14+ with the `psql` CLI on your PATH (`psql --version`).
  - (Optional) Node.js if you plan to run the React frontend.

2. **Import the Maven project**
   - Clone or unzip the repository.
   - Open `auction-platform/pom.xml` in your IDE (Eclipse or IntelliJ) using “Import existing Maven project” so the wrapper (`mvnw`) downloads dependencies.

3. **Configure PostgreSQL**
   - Create the application database:
     ```powershell
     psql -U postgres -c "CREATE DATABASE auction;"
     ```
   - Apply schema and seed data using the provided scripts:
     ```powershell
     psql -d auction -f scripts/create_schema.sql
     psql -d auction -f scripts/seed_sample_data.sql
     ```

4. **Set environment variables**
   - Create an `.env` file beside `auction-platform/pom.xml` (or configure OS-level variables) with:
     - `DB_URL` (default `jdbc:postgresql://localhost:5432/auction`)
     - `DB_USERNAME`
     - `DB_PASSWORD`
     - `JWT_SECRET_KEY` and `FORGOT_PASSWORD_SECRET` (Base64-encoded secrets)

5. **Build and run automated tests**
   ```powershell
   cd auction-platform
   ./mvnw.cmd clean test
   ```
   - Confirms the project compiles and unit tests pass.

6. **Start the Spring Boot application**
   - Package a runnable JAR: `./mvnw.cmd clean package`
   - Run from the packaged artifact: `java -jar target/auction-platform-0.0.1-SNAPSHOT.jar`
   - Or run directly for development: `./mvnw.cmd spring-boot:run`
   - Wait for `Started AuctionPlatformApplication` before calling REST endpoints on `http://localhost:8080`.

Stop the server with `Ctrl+C` when finished. Re-run the schema scripts whenever your local database drifts from the expected structure.

---
## 🔍 Automated Tests

- **How to run**: execute `./mvnw.cmd clean test` from `auction-platform/` to rebuild the backend and run all unit tests.
- **AuctionServiceTest** (`src/test/java/com/eecs4413/auction_platform/service/AuctionServiceTest.java`): covers auction search filters, detail lookups, and that a winning bid updates prices and emits WebSocket payloads.
- **PaymentServiceTest** (`src/test/java/com/eecs4413/auction_platform/service/PaymentServiceTest.java`): validates payment summaries, enforces price calculations, and checks guarded failure paths for invalid receipts.
- **UserServiceTest** (`src/test/java/com/eecs4413/auction_platform/service/UserServiceTest.java`): exercises registration, duplicate email rejection, login token issuance, and logout token revocation workflows.
- **PasswordResetServiceTest** (`src/test/java/com/eecs4413/auction_platform/service/PasswordResetServiceTest.java`): verifies forgot-password requests, reset token lifecycle, and password updates with attempt limits.
- **Troubleshooting**: if a context load fails due to schema drift, reapply `scripts/create_schema.sql` and `scripts/seed_sample_data.sql` so Hibernate’s validation aligns with the expected columns (e.g., `payments.paymentid`).

---
