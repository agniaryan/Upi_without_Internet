# UPI Offline Mesh
### Offline Payment Relay Simulation using Spring Boot, Java & Cryptography

What happens when two people want to transfer money but neither of them has an internet connection?

**UPI Offline Mesh** is a backend project that explores this problem by simulating an offline payment network where transactions can travel between nearby devices and reach a server later, when one of the devices regains internet access.

The project uses a Bluetooth-style mesh simulation to relay encrypted payment packets between virtual phones. Once an internet-connected device receives the packet, it forwards it to the backend, where the transaction is validated, duplicate requests are handled, and the payment is settled.

The entire flow runs locally on a single machine, so there's no need for actual Bluetooth hardware.

---

## How it works

Imagine you're in a basement with no network connection. You want to send ₹500 to a friend.

Instead of requiring an immediate internet connection, the payment is packaged into an encrypted message and passed between nearby devices. The message keeps moving through the network until a device with internet access can deliver it to the server.

The backend then processes the payment.

**The important part:** intermediate devices can carry the payment, but they cannot read or modify its contents.

## Demo workflow

<details>
<summary>Step 1 — Create a payment</summary>

Select a sender, receiver, amount and PIN on the dashboard. When you click **Inject into Mesh**, the backend creates a payment instruction, encrypts it and sends it to the first simulated device.

</details>

<details>
<summary>Step 2 — Relay the packet between devices</summary>

Click **Run Gossip Round** to simulate devices sharing packets with other nearby devices.

Each device forwards packets it has received, while the packet's TTL (Time To Live) decreases to limit how far it can travel.

</details>

<details>
<summary>Step 3 — Deliver the payment to the server</summary>

When **Flush Bridges** is clicked, devices with internet access upload the packets they have collected to the backend.

The server decrypts the packet, checks whether it is valid and recent, and attempts to settle the payment.

</details>

<details>
<summary>Step 4 — Handle duplicate deliveries</summary>

The same payment may reach the server through multiple devices. The backend ensures that receiving the same payment more than once does not result in multiple debits.

You can also run a concurrency test that sends the same packet through three threads simultaneously to verify this behavior.

</details>

## Tech stack

| Technology | Purpose |
|---|---|
| Java 17 | Backend development |
| Spring Boot 3.3 | REST APIs and application structure |
| Spring Data JPA | Database operations |
| H2 | In-memory database for the demo |
| AES-256-GCM | Encrypting payment data and detecting tampering |
| RSA-OAEP | Securely encrypting the AES key |
| Maven | Build and dependency management |
| HTML, CSS, JavaScript | Interactive dashboard |

## Architecture

The system is divided into four main parts:

- **Payment creation:** Generates the payment instruction and encrypts its contents.
- **Mesh simulation:** Virtual devices exchange packets through gossip rounds.
- **Bridge ingestion:** Internet-connected devices forward packets to the backend.
- **Settlement:** Validates payments and updates account balances and transaction records.

```text
Sender
   |
   | Creates & encrypts payment
   v
Virtual Phone
   |
   | Gossip / packet forwarding
   v
Nearby Devices
   |
   | Packet reaches an internet-connected device
   v
Bridge Node
   |
   | HTTP POST
   v
Spring Boot Backend
   |
   |-- Deduplication
   |-- Decryption & validation
   |-- Replay protection
   |
   v
Payment Settlement
   |
   |-- Debit sender
   |-- Credit receiver
   |-- Save transaction
```

## Technical challenges I worked on

### 1. Protecting payments while they travel through other devices

Mesh devices should not have access to the sender, receiver, or payment details. I implemented hybrid encryption using RSA-OAEP and AES-256-GCM.

Instead of encrypting the entire payment directly with RSA:

- A new AES key is generated for each payment.
- AES-GCM encrypts the payment data.
- RSA-OAEP encrypts the AES key using the server's public key.
- Both are packaged into a single ciphertext that the backend can decrypt.

AES-GCM also verifies the integrity of the encrypted data. If the ciphertext is modified, the authentication check fails during decryption.

### 2. Preventing the same payment from being processed twice

Multiple bridge devices may upload the same packet at nearly the same time. Processing every request independently could debit the sender multiple times.

To handle this, I implemented an idempotency mechanism using `ConcurrentHashMap.putIfAbsent()`.

Before decryption, the backend calculates a SHA-256 hash of the ciphertext and attempts to claim it. Only the first request is allowed to continue; subsequent requests are rejected as duplicates.

I also added a unique constraint on the transaction's packet hash as an additional safeguard.

### 3. Rejecting replayed or tampered packets

The backend checks the payment timestamp to reject packets older than 24 hours.

Each payment also contains a unique nonce, so two separate payments of the same amount are treated as different transactions, while retransmitting the same packet doesn't create another payment.

The encryption and idempotency checks work together to protect the settlement process.

## Project structure

```text
upi-offline-mesh/
│
├── src/main/java/com/demo/upimesh/
│   ├── controller/
│   │   ├── ApiController.java
│   │   └── DashboardController.java
│   │
│   ├── crypto/
│   │   ├── ServerKeyHolder.java
│   │   └── HybridCryptoService.java
│   │
│   ├── model/
│   │   ├── Account.java
│   │   ├── Transaction.java
│   │   ├── MeshPacket.java
│   │   └── PaymentInstruction.java
│   │
│   ├── service/
│   │   ├── DemoService.java
│   │   ├── MeshSimulatorService.java
│   │   ├── IdempotencyService.java
│   │   ├── SettlementService.java
│   │   ├── BridgeIngestionService.java
│   │   └── VirtualDevice.java
│   │
│   └── config/
│       └── AppConfig.java
│
└── src/test/
    └── IdempotencyConcurrencyTest.java
```

## Running locally

### Requirements

- Java 17+
- Windows, macOS or Linux

No database installation or separate service is required. The project uses H2 and the Maven Wrapper.

**Windows**

```bash
mvnw.cmd spring-boot:run
```

**macOS / Linux**

```bash
./mvnw spring-boot:run
```

Once the application starts, open:

[http://localhost:8080](http://localhost:8080)

### Run tests

```bash
mvnw.cmd test
```

Run the concurrency test separately:

```bash
mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

## API endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | Opens the demo dashboard |
| GET | `/api/server-key` | Returns the server's public RSA key |
| GET | `/api/accounts` | Returns account balances |
| GET | `/api/transactions` | Returns recent transactions |
| GET | `/api/mesh/state` | Shows the state of simulated devices |
| POST | `/api/demo/send` | Creates and injects an encrypted payment |
| POST | `/api/mesh/gossip` | Runs a mesh forwarding round |
| POST | `/api/mesh/flush` | Uploads packets from bridge devices |
| POST | `/api/mesh/reset` | Resets the mesh simulation |
| POST | `/api/bridge/ingest` | Receives and processes bridge packets |

## Testing

The project includes tests for the most important parts of the payment flow:

- **Encryption round-trip:** Verifies that encrypted payment data can be decrypted correctly.
- **Tampered packet:** Verifies that modified ciphertext is rejected.
- **Concurrent duplicate delivery:** Sends the same payment through three simultaneous requests and checks that only one transaction is settled.

## Limitations & future improvements

This project is a simulation of offline payment forwarding, not a real UPI integration.

Some important limitations:

- Payments are settled only when the packet reaches the backend. Until then, the receiver cannot independently confirm that the sender has sufficient funds.
- Offline users could attempt to send the same balance to multiple people. The backend can reject duplicate or invalid settlements later, but it cannot guarantee offline spending limits with this design.
- Mesh communication is simulated in software. Real Bluetooth Low Energy (BLE) or Wi-Fi Direct communication would require a mobile application and device-level networking.
- The current idempotency cache is in-memory and works within a single backend instance. A distributed deployment would need something like Redis and database-level safeguards.
- The demo generates its RSA key pair at startup. Production would require secure key storage, authenticated bridge nodes, user verification, rate limiting and integration with a regulated payment system.

## What I learned

Building this project helped me explore how backend systems handle requests when connectivity is unreliable and messages can be delivered more than once.

It gave me hands-on experience with Spring Boot services, cryptographic operations, transaction handling, concurrency, idempotency and testing failure scenarios — particularly how to make a payment-processing flow safer when the same request can arrive multiple times.

---

**Disclaimer:** This is an educational project that simulates mesh-routed, deferred payment settlement. It is not connected to UPI, NPCI, banks or real payment networks.
