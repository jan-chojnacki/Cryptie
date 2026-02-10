# Cryptie | End-to-End Encrypted Messaging System

A cross-platform, zero-knowledge messaging architecture utilizing hybrid RSA/AES cryptography and real-time signal propagation to ensure confidentiality, integrity, and non-repudiation of communications.

## 📺 Demo & Visuals
*Visual representation of the system in operation.*

### 🔐 Authentication & Security Onboarding
*Multifactor authentication flow and cryptographic identity initialization.*

* **Registration & Login:** ![Register](/docs/screenshots/Register.png) ![Login](/docs/screenshots/Login.png)
* **Identity Protection (MFA & PIN):** ![Qr Code](/docs/screenshots/Qr%20Code.png) ![One Time Password](/docs/screenshots/One%20Time%20Password.png) ![Pin Code](/docs/screenshots/Pin%20Code.png)
* **System State:** ![Loading](/docs/screenshots/Loading.png)

### 💬 Core Interface & User Experience
*Cross-platform UI utilizing Avalonia for native-like performance and responsive design.*

* **Main Dashboard:** ![Home Screen - No Friends](/docs/screenshots/Home%20Screen%20-%20No%20Friends.png)
* **Theme Support (Dark Mode):** ![Home Screen - Dark Mode](/docs/screenshots/Home%20Screen%20-%20Dark%20Mode.png)
* **Account & Configuration:** ![Account](/docs/screenshots/Account.png) ![Settings](/docs/screenshots/Settings.png)

## 🏗️ Architecture & Context
*High-level system design and structural logic.*

* **Objective:** Provision of a secure communication platform where the server functions as a stateless relay, possessing zero access to plaintext message content or user private keys.
* **Architecture Pattern:**
    * **Client:** MVVM (Model-View-ViewModel) architecture leveraging ReactiveUI for state management and Avalonia for cross-platform rendering.
    * **Server:** Service-Oriented Architecture (SOA) utilizing ASP.NET Core for resource management and SignalR for real-time event distribution.
* **Data Flow:**
    1.  **Identity:** RSA-2048 key pairs are generated locally. The public key is registered on the server; the private key is encrypted via AES (PIN-derived hash) and persisted in the OS Keychain.
    2.  **Key Exchange:** Session keys (AES-256) are generated per conversation and exchanged using the recipient's RSA public key.
    3.  **Transport:** Payloads are encrypted locally and transmitted via SignalR. The server persists ciphertext in PostgreSQL for asynchronous delivery but remains incapable of decryption.

## ⚖️ Design Decisions & Trade-offs
*Technical justifications for architectural choices.*

* **Cryptography: RSA-2048 over ECC (Elliptic Curve Cryptography)**
    * **Context:** Selection of an asymmetric primitive for identity and key encapsulation that must interface with hardware-backed security modules.
    * **Decision:** Implementation of RSA-2048 utilizing PKCS#1 v2.1 (OAEP) padding.
    * **Rationale:** RSA-2048 was selected to leverage hardware-accelerated key generation within TPM 2.0 modules and Secure Enclaves. Many consumer-grade TPMs lack native, hardware-isolated support for Ed25519, making RSA the only viable option for ensuring private keys are non-exportable from the silicon level.
    * **Trade-off:** Accepted significantly larger key sizes and higher computational latency compared to ECC, which directly necessitated a custom segmentation strategy to bypass OS-level storage limits for cryptographic assets.

* **Persistence: OS-Native Secure Storage**
    * **Context:** Protection of private keys against local exfiltration by malicious processes.
    * **Decision:** Offloading key persistence to hardware-backed modules (Windows Credential Manager, macOS Keychain, Linux Keyring).
    * **Rationale:** Ensures that cryptographic assets are managed by the operating system's security kernel rather than the application’s file system.
    * **Trade-off:** Increased implementation complexity due to platform-specific storage quotas (e.g., the 2560-byte limit on certain Windows versions), requiring a chunking abstraction layer.

* **Relay Strategy: Stateless Server-side Persistence**
    * **Context:** Balancing high availability with zero-knowledge requirements.
    * **Decision:** The server persists encrypted blobs until delivery is acknowledged by the recipient.
    * **Rationale:** Decouples message delivery from client uptime, allowing for reliable asynchronous communication.
    * **Trade-off:** The server stores metadata (sender/receiver IDs and timestamps), which is a necessary compromise to facilitate message routing without decryption.

## 🧠 Engineering Challenges
*Analysis of non-trivial technical hurdles and resolutions.*

* **Challenge: Ensuring Causal Consistency in Asynchronous Delivery**
    * **Problem:** Messages arriving via different transport paths (SignalR vs. REST history) or during intermittent connectivity frequently result in "out-of-order" rendering or missing conversation gaps.
    * **Implementation:** Developed a **Hash-Linked Sequencing** mechanism. Each message contains a monotonically increasing sequence number and the hash of the preceding message. The client-side logic verifies this chain upon receipt.
    * **Outcome:** The system automatically detects gaps in the message chain. If a sequence break is identified, the client triggers a selective re-synchronization request for the missing indices, ensuring total causal ordering without server-side knowledge of the conversation state.

* **Challenge: Mitigating Side-Channel Timing Attacks**
    * **Problem:** Discrepancies in execution time during authentication (e.g., verifying a hash vs. immediate rejection for non-existent users) allow attackers to enumerate valid usernames.
    * **Implementation:** Implementation of a `DelayService` that enforces a constant-time floor for all security-sensitive operations. The service measures execution duration and injects precise micro-delays to normalize response latency.
    * **Outcome:** Statistical distribution of response times is identical regardless of input validity, neutralizing username enumeration through timing analysis.

* **Challenge: Handling Hardware Storage Quotas (Keychain Chunking)**
    * **Problem:** Platform-specific credential managers enforce strict size limits that are insufficient for full RSA private key assets and metadata.
    * **Implementation:** Engineered a `KeychainManagerService` that segments large secrets into 1KB chunks. A primary "Meta" entry tracks chunk count and parity, while individual segments are distributed across sequential slots.
    * **Outcome:** Reliable persistence of cryptographic material across Windows, macOS, and Linux without triggering platform-specific overflow exceptions.

## 🛠️ Tech Stack & Ecosystem
* **Core:** .NET 9 (C#)
* **Client UI:** Avalonia UI, ReactiveUI
* **Server Runtime:** ASP.NET Core 9, Docker
* **Persistence:** PostgreSQL, Entity Framework Core
* **Communication:** SignalR (WebSockets), HTTP/2
* **Cryptography:** BouncyCastle, System.Security.Cryptography
* **Infrastructure:** Testcontainers (Integration Testing)

## 🧪 Quality & Standards
* **Testing Strategy:**
    * **Unit Testing:** xUnit for isolation of cryptographic primitives and business logic.
    * **Integration Testing:** **Testcontainers** is utilized to deploy ephemeral PostgreSQL instances, ensuring validation against a production-grade schema during CI/CD.
* **Engineering Principles:**
    * **Zero Knowledge:** Architecture enforces client-side encryption; keys are never transmitted to or stored on the server.
    * **Security by Design:** Mandatory 2FA (TOTP) and constant-time execution floors for all authentication vectors.

## 🙋‍♂️ Authors

**Kamil Fudala**

- [GitHub](https://github.com/FreakyF)
- [LinkedIn](https://www.linkedin.com/in/kamil-fudala/)

**Jan Chojnacki**

- [GitHub](https://github.com/Jan-Chojnacki)
- [LinkedIn](https://www.linkedin.com/in/jan-chojnacki-772b0530a/)

**Jakub Babiarski**

- [GitHub](https://github.com/JakubKross)
- [LinkedIn](https://www.linkedin.com/in/jakub-babiarski-751611304/)

## ⚖️ License

This project is licensed under the [MIT License](LICENSE).
