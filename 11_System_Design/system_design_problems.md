# System Design — Common Problems

> End-to-end architecture designs for classic MAANG system design interviews. Includes High-Level, Component, and Data-Flow diagrams.

---

## Table of Contents

- [1. Design a URL Shortener (TinyURL)](#1-design-a-url-shortener-tinyurl)
- [2. Design a Chat Application (WhatsApp/Discord)](#2-design-a-chat-application-whatsappdiscord)
- [3. Design a Video Streaming Service (Netflix/YouTube)](#3-design-a-video-streaming-service-netflixyoutube)

---

## 1. Design a URL Shortener (TinyURL)

**Requirements:**
- Given a long URL, return a much shorter unique alias.
- When users click the short alias, redirect them to the original URL.
- Links expire after standard timesspan.
- Highly available and scalable (Read-heavy: 100:1 read/write ratio).

### High-Level Architecture

```mermaid
graph TD
    Client((Client)) -->|Write: POST /create| API_GW[API Gateway / Load Balancer]
    Client -->|Read: GET /xyz123| API_GW
    
    API_GW --> WebServers[Web Servers]
    
    WebServers -->|Cache Miss| Cache[(Redis Cache)]
    WebServers -->|Write / Cache Miss| DB[(NoSQL DB / Cassandra)]
    
    WebServers -->|Get Unique ID| KGS[Key Generation Service]
```

### Component Details: Key Generation Service (KGS)

We need to generate a unique 7-character string (Base62: A-Z, a-z, 0-9). `62^7 = 3.5 trillion` URLs.
Generating keys on the fly using a hash (like MD5) can cause collisions. Instead, we pre-generate them.

```mermaid
graph LR
    subgraph KGS Cluster
    KGS1[KGS Node 1]
    KGS2[KGS Node 2]
    end
    
    KGS1 -->|Fetch block of unused keys| KGS_DB[(Key DB)]
    KGS2 -->|Fetch block of unused keys| KGS_DB
    
    KGS1 -->|Provide Key| WebServer1[Web Server]
```
*The KGS pre-generates random 7-char strings and stores them in a DB. It loads a batch into memory. When a web server needs a key, it asks the KGS. This avoids DB latency and guarantees uniqueness.*

### Data Flow: Redirection (Read)

```mermaid
sequenceDiagram
    participant User
    participant LB as Load Balancer
    participant App as App Server
    participant Cache as Redis Cache
    participant DB as NoSQL DB

    User->>LB: GET /xyz123
    LB->>App: Route request
    App->>Cache: Check key "xyz123"
    
    alt Cache Hit
        Cache-->>App: Return Long URL
    else Cache Miss
        App->>DB: Query key "xyz123"
        DB-->>App: Return Long URL
        App->>Cache: Update Cache
    end
    
    App-->>User: HTTP 301 Redirect (Location: Long URL)
```
*Note: We use **HTTP 301 (Permanent Redirect)** if we want the browser to cache the redirect to reduce server load. We use **HTTP 302 (Temporary Redirect)** if we need to track analytics for every single click.*

---

## 2. Design a Chat Application (WhatsApp/Discord)

**Requirements:**
- 1-on-1 messaging with low latency.
- Online/Offline presence indicator.
- Message persistence (sync across devices).

### High-Level Architecture

```mermaid
graph TD
    Client1((User A)) <-->|WebSocket| LB[Load Balancer]
    Client2((User B)) <-->|WebSocket| LB
    
    LB <--> ChatServers[Chat Servers]
    
    ChatServers --> PresenceService[Presence Service]
    ChatServers --> PushNotif[Push Notification Service]
    
    ChatServers --> MessageQueue[Message Queue / Kafka]
    MessageQueue --> DBWorker[Database Workers]
    DBWorker --> Cassandra[(Cassandra / Wide-Column DB)]
    
    ChatServers <--> RedisPubSub[(Redis Pub/Sub)]
```

### Component Details: Stateful Connection Management

Unlike standard HTTP APIs, chat requires **persistent WebSocket connections**. The system must know exactly which server User B is connected to so it can route User A's message to them.

```mermaid
graph TD
    subgraph Service Discovery / Routing
    UserA -->|Send msg to User B| ChatServer1[Chat Server 1]
    ChatServer1 -->|Where is B?| RedisSession[(Redis Session Cache)]
    RedisSession -->>|B is on Server 3| ChatServer1
    ChatServer1 -->|Forward Msg| ChatServer3[Chat Server 3]
    ChatServer3 -->|Push over WS| UserB
    end
```

### Data Flow: Message Delivery

```mermaid
sequenceDiagram
    participant A as User A
    participant S1 as Chat Server 1
    participant Session as Session Cache
    participant S2 as Chat Server 2
    participant B as User B
    participant DB as Cassandra DB

    A->>S1: Send "Hello" to B (WebSocket)
    S1->>DB: Async save message (via Kafka)
    S1->>Session: Get User B's server
    
    alt B is Online
        Session-->>S1: User B is on Server 2
        S1->>S2: Forward "Hello"
        S2->>B: Push "Hello" (WebSocket)
        B-->>S2: ACK receipt
        S2-->>S1: ACK receipt
        S1-->>A: Show "Delivered" tick
    else B is Offline
        Session-->>S1: User B is offline
        S1->>PushNotif: Trigger Apple/Google Push Notification
    end
```

---

## 3. Design a Video Streaming Service (Netflix/YouTube)

**Requirements:**
- Upload videos, process them, and store them.
- Stream videos globally with minimal buffering.
- Search for videos.

### High-Level Architecture

```mermaid
graph TD
    User((User)) -->|Search / Browse| API[API Gateway]
    API --> MetadataDB[(Metadata DB / Postgres)]
    API --> ElasticSearch[(ElasticSearch)]
    
    User -->|1. Upload| UploadService[Upload Service]
    UploadService --> OriginalS3[(S3: Raw Videos)]
    OriginalS3 -->|Trigger Event| MessageQueue[Kafka]
    
    MessageQueue --> EncodingWorkers[Encoding Workers]
    EncodingWorkers --> TranscodedS3[(S3: Transcoded Videos)]
    
    TranscodedS3 -->|Replicate| CDN[Global CDN]
    User -->|2. Stream| CDN
```

### Component Details: Video Processing Pipeline

Video files are massive. When a user uploads a 4K video, it cannot be streamed directly. It must be chunked and transcoded into different resolutions (1080p, 720p, 360p) and formats (HLS, DASH) for different devices and network speeds.

```mermaid
graph LR
    RawFile[Raw Video File] --> Chunking[Chunking Service]
    Chunking -->|Chunk 1| EW1[Encoding Worker]
    Chunking -->|Chunk 2| EW2[Encoding Worker]
    Chunking -->|Chunk N| EW3[Encoding Worker]
    
    EW1 -->|1080p, 720p, 480p| Merger[Merger / Manifest Generator]
    EW2 -->|1080p, 720p, 480p| Merger
    EW3 -->|1080p, 720p, 480p| Merger
    
    Merger --> HLS[HLS Manifest File]
    Merger --> S3[(Final Storage)]
```
*Chunking allows parallel processing. A 1-hour video can be split into 1-minute chunks, encoded by 60 different workers simultaneously, drastically reducing upload-to-publish time.*

### Data Flow: Streaming & Adaptive Bitrate

How does Netflix ensure you don't buffer when you enter a tunnel?

```mermaid
sequenceDiagram
    participant App as Client App
    participant CDN as Content Delivery Network
    
    App->>CDN: GET Manifest File (index.m3u8)
    CDN-->>App: Manifest (Links to 1080p, 720p, 360p chunks)
    
    Note over App: Network is strong (WiFi)
    App->>CDN: GET Chunk 1 (1080p)
    CDN-->>App: Video Data
    
    Note over App: Network drops (Cellular 3G)
    App->>CDN: GET Chunk 2 (360p)
    CDN-->>App: Video Data
    
    Note over App: Network recovers
    App->>CDN: GET Chunk 3 (720p)
    CDN-->>App: Video Data
```
*This is called **Adaptive Bitrate Streaming (ABR)**. The client downloads a tiny text file (manifest) containing links to 5-second video chunks at various resolutions. The client software decides which resolution to request next based on its current bandwidth.*

---

*End of System Design Problems — Practical end-to-end architectures utilizing components like CDNs, WebSockets, Kafka, and Redis.*
