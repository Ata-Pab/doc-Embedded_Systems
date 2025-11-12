# 🌐 Visual Network Flow Diagrams for IoT Communication

From **low-level physical connection (Wi-Fi)** up to **high-level secure communication (HTTPS / MQTT)**.


## 1. Wi-Fi Connection Flow

When your ESP32 connects to Wi-Fi, it becomes a node in your local network.

```
   ┌──────────────────────┐
   │       ESP32          │
   │(Station Mode - STA)  │
   └─────────┬────────────┘
             │  Wi-Fi (802.11)
             │  SSID: "MyHomeWiFi"
             │  Pass: "12345678"
             ▼
   ┌──────────────────────┐
   │      Wi-Fi Router    │
   │ (Access Point - AP)  │
   └─────────┬────────────┘
             │  Ethernet / Fiber
             ▼
   ┌──────────────────────┐
   │     Internet (ISP)   │
   └─────────┬────────────┘
             ▼
   ┌──────────────────────┐
   │   Remote Server /    │
   │   Cloud Service      │
   └──────────────────────┘
```

### Sequence:

1. ESP32 sends **Probe Requests** → finds AP (router).
2. Performs **Authentication** and **Association**.
3. Gets **IP address via DHCP** from router.
4. Now it can send packets on the local network and out to the Internet.

---

## 2. IP Layer (Network Layer)

After connecting, the ESP32 has an IP (e.g., `192.168.1.105`).
Your router has its own WAN IP (e.g., `85.105.22.6`) given by your ISP.

When you access a remote server:

```
ESP32 (192.168.1.105) → Router (NAT) → Internet → Server (104.26.12.52)
```

The router translates addresses using **NAT (Network Address Translation)**.

📘 **Key idea:**
Your ESP32 doesn’t know the Internet directly — it sends packets to the router, and the router forwards them to the world.

---

## 3. DNS Resolution (Finding the Server’s IP)

Before sending data to `https://httpbin.org`, the ESP32 must find its **IP address** via DNS.

```
┌──────────────────────┐
│     ESP32            │
└───────┬──────────────┘
        │ "What is the IP of httpbin.org?"
        ▼
┌──────────────────────┐
│  DNS Server (e.g.,   │
│  8.8.8.8 by Google)  │
└───────┬──────────────┘
        │ "104.26.12.52"
        ▼
┌──────────────────────┐
│ ESP32 now knows IP   │
└──────────────────────┘
```

This happens automatically in ESP-IDF when you pass a hostname to the HTTP client.

---

## 4. TCP Handshake (Connection Setup)

Before any data is exchanged, a **TCP connection** must be established — think of it as a reliable “pipe” between ESP32 and the server.

```
ESP32                                Server
  │ -------- SYN ------------> │   (I want to start connection)
  │ <------- SYN-ACK --------- │   (Okay, let's sync)
  │ -------- ACK ------------> │   (Confirmed)
  │──────── Connected! ───────>│
```

Now the devices can exchange reliable packets with guaranteed delivery and order.

---

## 5. TLS Handshake (Securing the Pipe)

Next, the **TLS (Transport Layer Security)** handshake builds an encrypted tunnel over that TCP pipe:

```
ESP32                                        Server
  │ -------- ClientHello ------------> │  (I want to use TLS)
  │ <------- ServerHello + Cert -------│  (Here’s my certificate)
  │ -------- Verify Cert --------------│  (Check CA root PEM)
  │ -------- Key Exchange ------------>│  (Generate session key)
  │ <------- Encrypted Ready ----------│
  │ >>> All Data Encrypted >>>>>>>>>>> │
```

💡 After this, everything — HTTP headers, JSON data, even URLs — is encrypted.
No one (not even the router) can see your actual data.

---

## 6. HTTP Request & Response Flow (Over TLS)

Now the ESP32 sends the actual **HTTP POST** with JSON data:

```
ESP32                                       Cloud Server
  │                                        │
  │ POST /data HTTP/1.1                    │
  │ Host: httpbin.org                      │
  │ Content-Type: application/json         │
  │                                        │
  │ {"temp":25.3, "humidity":44} --------->│
  │                                        │
  │                        <---------------│
  │ HTTP/1.1 200 OK                        │
  │ {"message": "data received"}           │
```

All of this is **inside the TLS-encrypted session**.

---

## 7. FreeRTOS Task Interaction in ESP32

Here’s how your **firmware tasks** coordinate this network flow:

```
┌─────────────────────┐
│ Wi-Fi Task          │
│ Connect to network  │
│ Give semaphore      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Producer Task       │
│ Create JSON payload │
│ Send to Queue       │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ HTTP Task           │
│ Wait for Queue item │
│ POST via HTTPS      │
│ Handle response     │
└─────────────────────┘
```

This architecture lets:

* The system stay responsive
* Data flow smoothly
* Network events be handled asynchronously

---

## 8. Full IoT Data Path Overview

```
        ┌──────────────────────────┐
        │        ESP32 Device      │
        │ ────────────────         │
        │ Wi-Fi → TCP → TLS → HTTP │
        │ JSON: {"temp":25.3}      │
        └──────────┬───────────────┘
                   │
                   ▼
        ┌──────────────────────────┐
        │   Wi-Fi Router / NAT     │
        │ (Translates IP packets)  │
        └──────────┬───────────────┘
                   │
                   ▼
        ┌──────────────────────────┐
        │     Internet Backbone    │
        │   (Routers, DNS, etc.)   │
        └──────────┬───────────────┘
                   │
                   ▼
        ┌──────────────────────────┐
        │   Cloud Server / API     │
        │  https://httpbin.org     │
        │  → Parse JSON, respond   │
        └──────────────────────────┘
```

---

## 9. Response Handling & Disconnection Recovery

If the network drops:

* Wi-Fi task notices → takes semaphore.
* HTTP task suspends (waits).
* Once reconnected → semaphore given again.
* HTTP resumes sending new data.

This separation keeps **tasks independent** and **system stable** — a key design goal in professional IoT firmware.

---

## 🧠 Summary

1. ESP32 connects to Wi-Fi (gets IP).
2. Resolves server IP via DNS.
3. Opens TCP connection.
4. Performs TLS handshake → secure channel.
5. Sends HTTP POST with JSON payload.
6. Receives HTTP response.
7. Closes connection or keeps alive.
8. FreeRTOS synchronizes tasks via queue/semaphore.

---
