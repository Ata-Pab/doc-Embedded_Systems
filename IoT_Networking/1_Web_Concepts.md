# 🌐 IoT Networking & Web Concepts — From Ground Up

## 1️. The Internet Stack — Layers

In modern IoT systems, communication happens through layers — each with a specific responsibility.
We typically refer to the **OSI model (7 layers)** or the **simplified TCP/IP model (4 layers)**:

| Layer           | TCP/IP Equivalent | Example in ESP32 |
| --------------- | ----------------- | ---------------- |
| 7 – Application | Application       | HTTP, MQTT, DNS  |
| 4 – Transport   | Transport         | TCP, UDP         |
| 3 – Network     | Internet          | IP (IPv4/IPv6)   |
| 2 – Data Link   | Network Interface | Wi-Fi driver     |
| 1 – Physical    | Hardware          | Radio / Ethernet |

📘 **In short:**
ESP32 connects to Wi-Fi (Layer 2) → gets an IP (Layer 3) → sends data via TCP or UDP (Layer 4) → uses HTTP or MQTT (Layer 7).

---

## 2. TCP vs UDP — Transport Layer

These are the **two main transport protocols** on the Internet. They define *how data travels between two devices*.

| Feature     | TCP (Transmission Control Protocol) | UDP (User Datagram Protocol)    |
| ----------- | ----------------------------------- | ------------------------------- |
| Connection  | Connection-oriented (handshake)     | Connectionless                  |
| Reliability | Reliable (retransmits lost data)    | Unreliable (no retry)           |
| Order       | Maintains packet order              | Packets may arrive out of order |
| Speed       | Slower, more overhead               | Faster, lightweight             |
| Use Cases   | HTTP, HTTPS, MQTT                   | DNS, video streaming, gaming    |

💡 In IoT, **TCP** is used for reliability (e.g., HTTP/MQTT), while **UDP** is used for real-time low-latency (e.g., sensors or telemetry bursts).

---

## 3. IP Address, Ports, and Sockets

Once a device is on a network (e.g., via Wi-Fi), it receives an **IP address** like `192.168.1.105`.
Communication between two devices happens via **sockets**, defined by:

```
<IP address>:<port number>
```

Example:

```
192.168.1.105:5555 → 104.26.12.52:80
```

* Port **80** → HTTP
* Port **443** → HTTPS
* Port **1883** → MQTT
* Port **8883** → MQTT over TLS

📘 In ESP-IDF, a socket is represented internally as a handle in `lwIP` (the TCP/IP stack ESP32 uses).

---

## 4. HTTP — HyperText Transfer Protocol

HTTP defines **how clients (ESP32, browsers, etc.) talk to servers**.
It’s the most common “application-layer protocol” for IoT and the Web.

A **client** sends a **request**, and the **server** replies with a **response**.

Example:

```
Client → POST /data HTTP/1.1
Host: httpbin.org
Content-Type: application/json
Content-Length: 29

{"temp": 24.6, "humidity": 43}
```

Server Response:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 123

{"message":"data received","status":200}
```

🧠 HTTP defines:

* **Methods**: `GET`, `POST`, `PUT`, `DELETE`, etc.
* **Headers**: key-value metadata
* **Body**: optional data (e.g., JSON payload)

💡 ESP-IDF’s `esp_http_client` handles all this — it wraps socket handling, packet assembly, and parsing.

---

## 5. HTTPS — Secure HTTP

HTTP + TLS encryption layer = **HTTPS**.

It ensures:

1. **Confidentiality** → Data is encrypted.
2. **Integrity** → No one can modify packets undetected.
3. **Authentication** → You’re sure you’re talking to the real server (not a fake one).

All of this is provided by **TLS** (Transport Layer Security), using **certificates**.

---

## 6. Certificates & PEM Files

A **certificate** is a digital file that identifies a server and enables encryption.
It’s signed by a trusted third party called a **CA (Certificate Authority)**.

### 📜 PEM Format

PEM = *Privacy Enhanced Mail* — just a Base64 text encoding of certificates or keys.

Example:

```
-----BEGIN CERTIFICATE-----
MIIFazCCA1OgAwIBAgIQD1Q1m0o...
-----END CERTIFICATE-----
```

A PEM can represent:

* A **root CA certificate** (trust anchor)
* A **server certificate**
* A **private key**

---

## 7. CA (Certificate Authority)

A **CA** is a trusted organization that verifies identities on the internet and issues certificates.
Examples:

* Let's Encrypt
* DigiCert
* GlobalSign

Each certificate forms part of a **chain of trust**:

```
Root CA → Intermediate CA → Server Certificate
```

ESP32 verifies the server certificate using the **Root CA PEM** you embedded.

If the certificate chain matches and signatures are valid → connection is trusted.

---

## 8. CA Bundle

A **CA bundle** is simply a collection of multiple root certificates.
Your computer (or sometimes IoT firmware) keeps this bundle to trust many websites.

For IoT devices:

* You can **embed only the root PEM** for your specific server (smaller, safer).
* Or embed a **bundle** (more flexible, bigger flash usage).

---

## 9. TLS Handshake (Simplified)

When you connect to `https://httpbin.org`, this happens:

1. **Client Hello** → ESP32 says "I want to connect securely. Here are my supported ciphers."
2. **Server Hello** → Server replies with its certificate.
3. **Certificate Validation** → ESP32 checks the certificate against the CA PEM.
4. **Key Exchange** → Both sides derive session keys.
5. **Encrypted Communication** → All data now goes through AES encryption.

All of this happens automatically inside ESP-IDF’s `esp-tls` and `esp_http_client`.

---

## 10. JSON — Data Format for HTTP/MQTT

JSON = JavaScript Object Notation, a lightweight way to encode structured data:

```json
{
  "device": "esp32",
  "temperature": 23.7,
  "humidity": 45
}
```

Why IoT loves JSON:

* Human-readable
* Small size
* Natively supported by REST APIs and web backends
* Easy to parse with `cJSON` library in ESP-IDF

---

## 11. MQTT (Preview)

MQTT = **Message Queue Telemetry Transport** — a publish/subscribe protocol designed for IoT.

It runs over **TCP (port 1883)** or **TLS (port 8883)** and is **lightweight**, compared to HTTP.

> HTTP = request/response

> MQTT = event-driven publish/subscribe

---

# 🧠 Summary Table

| Concept       | Description                       | Example                              |
| ------------- | --------------------------------- | ------------------------------------ |
| **TCP**       | Reliable, ordered data delivery   | HTTP, MQTT                           |
| **UDP**       | Fast, no guarantee                | Streaming                            |
| **HTTP**      | Request/response protocol         | `POST /data`                         |
| **HTTPS**     | HTTP + TLS encryption             | `https://api.server.com`             |
| **TLS**       | Security layer for encryption     | Used by HTTPS, MQTTs                 |
| **PEM**       | Text file containing certificate  | `root_ca.pem`                        |
| **CA**        | Trusted signer verifying identity | Let's Encrypt                        |
| **CA Bundle** | Multiple trusted roots            | Linux `/etc/ssl/certs/ca-bundle.crt` |
| **JSON**      | Structured data format            | `{"temp":25}`                        |

---

