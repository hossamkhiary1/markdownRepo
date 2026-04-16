# DoIP Server Implementation Design Document

## 1. Overview
The \DoIp.cpp\ module provides the core implementation of the **Diagnostic communication over Internet Protocol (DoIP)** server stack, adhering strictly to **ISO-13400**. It orchestrates TCP and UDP communications, handles vehicle announcements, and dynamically manages connected diagnostic testers through isolated client interfaces (\DoIpClientInterface\).

---

## 2. Architecture & Class Diagram

The architecture is heavily modularized. The primary \DoIp\ class manages the overarching lifecycle, server configuration, UDP communication, and thread pools. Each TCP diagnostic connection spawns a dedicated \DoIpClientInterface\ managing its specific session state, inactivity timers, and message dispatching.

\\mermaid
classDiagram
    class DoIp {
        -TcpInterfaceImpl m_tcpInterface
        -UdpInterface m_udpInterface
        -VehicleIdentifierHandler m_vinHandler
        +start() bool
        +stop() bool
    }
    class DoIpClientInterface {
        -DoIpStateHandler m_StateHandler
        -DoIpPayloadDispatcher m_PayloadDispatcher
        -DoIpTimerInterface m_DoIPInitialInactivityTimer
        +start()
        +stop()
    }
    class TcpInterfaceImpl
    class UdpInterface
    class VehicleIdentifierHandler
    class DoIpConfigManager
    class DoIpPayloadDispatcher

    DoIp --> TcpInterfaceImpl : Uses for TCP Server
    DoIp --> UdpInterface : Uses for UDP Broadcasts & Listening
    DoIp --> DoIpClientInterface : Instantiates & Manages
    DoIp --> VehicleIdentifierHandler : Aggregates VIN Data
    DoIp --> DoIpConfigManager : Fetches Configuration
    DoIpClientInterface --> DoIpPayloadDispatcher : Dispatches Requests
\
---

## 3. Sequence Diagrams

### 3.1. Server Startup and Sub-Systems Initialization
The server initialization process brings up the TCP and UDP network interfaces and starts background workers for connection management and vehicle announcements.

\\mermaid
sequenceDiagram
    participant App as Application
    participant DoIp as DoIp Server
    participant TCP as TcpInterfaceImpl
    participant UDP as UdpInterface
    participant Threads as Worker Threads
    
    App->>DoIp: start()
    DoIp->>TCP: startServer(ip, port)
    DoIp->>UDP: startServer(port)
    
    par Background Workers
        DoIp->>Threads: Spawn tcpClientAcceptanceFunction
        DoIp->>Threads: Spawn udpListeningFunction
        DoIp->>Threads: Spawn udpTransmitFunction
        opt If Announcement Enabled
            DoIp->>Threads: Spawn announcementTransmitFunction
        end
    end
    DoIp-->>App: true (Server Running)
\
### 3.2. Diagnostic Tester Connection & Message Dispatching
When a new tester connects, a \DoIpClientInterface\ is spawned. It runs concurrent threads for RX and TX to avoid blocking the main server.

\\mermaid
sequenceDiagram
    participant Tester as Diagnostic Tester
    participant Accept as DoIp::tcpClientAcceptanceFunction
    participant Client as DoIpClientInterface
    participant Dispatcher as DoIpPayloadDispatcher
    participant TCP as TcpInterfaceImpl
    
    Tester->>Accept: Connect Request (TCP)
    Accept->>Client: Instantiates & start()
    
    par Client Threads
        Client->>Client: runReceive() thread
        Client->>Client: runTransmit() thread
    end
    
    Tester->>Client: Send DoIP Message (e.g., Routing Activation)
    Client->>TCP: receiveExact() Header & Payload
    TCP-->>Client: Raw Bytes
    
    Client->>Client: receiveDoIpMessage() (Parsing)
    Client->>Client: handleDoIpReceivedRequest()
    Client->>Dispatcher: handlerDispatch(reqType, payload)
    Dispatcher-->>Client: Response Vector
    
    Client->>Client: createDoIpResponse()
    Client-->>Tester: Transmits Ack/Response via runTransmit()
\
---

## 4. API Reference

### 4.1. Core \DoIp\ Class
Serves as the central manager for the DoIP stack.

#### **Constructors & Lifecycle**
* **\DoIp()\** Constructs a new \DoIp\ server instance and fetches network and protocol settings.
* **\~DoIp()\** Destructs the \DoIp\ server instance and ensures graceful shutdown.
* **\ool start()\** Starts the DoIP Server TCP and UDP interfaces and spawns worker threads.
* **\ool stop()\** Stops the DoIP Server, closes all active connections, and joins threads.

#### **Worker Threads (Private)**
* **\oid tcpClientAcceptanceFunction()\** Continuously accepts incoming TCP client connections.
* **\oid udpListeningFunction()\** Listens for incoming UDP messages.
* **\oid udpTransmitFunction()\** Processes and transmits pending UDP responses.
* **\oid announcementTransmitFunction()\** Periodically broadcasts Vehicle Identification Response messages.

#### **Connection Management**
* **\oid handleDisconnectedClients()\** Cleans up disconnected TCP clients.
* **\oid removeDisconnectedClient(SOCKET clientSocket)\** Safely halts threads and removes a specifically disconnected socket.
* **\oid notifyClientDisconnected()\** Wakes the acceptance thread when maximum capacity was reached.

---

### 4.2. \DoIpClientInterface\ Class
Handles an individual TCP client connection, operating its own lifecycle, timers, and state machine.

#### **Client Lifecycle**
* **\oid start()\** Initializes state to \INIT\ and launches dedicated receive and transmit threads.
* **\oid stop()\** Safely tears down the client interface.

#### **Reception and Dispatch**
* **\ReceptionAction receiveDoIpMessage(...)\** Parses incoming DoIP headers from the socket.
* **\oid handleDoIpReceivedRequest(...)\** Dispatches parsed requests to their target implementation handler.
* **\oid createDoIpResponse(...)\** Constructs an outbound DoIP message array.

#### **ISO-13400 Timer Controls**
* **\oid startInitialInactivityTimer()\ / \stopInitialInactivityTimer()\** Manages the initial DoIP timer.
* **\oid startGeneralInactivityTimer()\ / \stopGeneralInactivityTimer()\** Manages the general inactivity timer.

#### **Security & Resource Checks**
* **\ool isAddressUsed(uint16_t logicalAddress)\** Thread-safe check across the server to prevent concurrent assignments.
