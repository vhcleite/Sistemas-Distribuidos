# UDP Message Delivery Demo

A small Java project created for the Distributed Systems course at UFABC. It demonstrates how an application can add a few delivery-oriented behaviors on top of UDP: packet sequencing, buffering, duplicate detection, acknowledgements, and a simple missing-packet recovery signal.

This is a learning exercise, not a production-ready reliable-UDP implementation.

## What it demonstrates

The bundled client starts two independent sender threads. Each sends a sequence of ten payloads (`M0` through `M9`) to a UDP server on `127.0.0.1:9876`.

Each payload is represented as:

```text
<sequence-number>:<data>:<total-packets>:
```

For example:

```text
3:M3:10:
```

The server replies with a structured status message whose data field is one of:

| Status | Meaning |
| --- | --- |
| `OK` | The packet was added to the client buffer. |
| `REPEATED_PACKET` | A packet with the same sequence number and data was already buffered. |
| `MISSING_PACKET` | The handler did not receive the expected number of packets before its timeout. |

The default client configuration exercises two scenarios concurrently:

- **Ordered messages** — sends packets in sequence.
- **Lost message** — deliberately skips packet index `5`; after the server's five-second timeout, the client receives `MISSING_PACKET` and resends it.

Additional scenarios are available in `TestCaseEnum`: unordered messages and a repeated message.

## Architecture

```text
                         UDP / 127.0.0.1:9876
+-------------------+  ----------------------->  +-----------------------+
| UDPClient         |                             | UDPServer             |
| ├─ ClientThread 1 |                             | └─ dispatcher loop     |
| └─ ClientThread 2 |  <-----------------------  |    └─ handler per      |
+-------------------+       acknowledgement       |       IP:port client   |
                                                    +-----------------------+
                                                              |
                                                              v
                                                    +-----------------------+
                                                    | ClientHandlerThread   |
                                                    | ├─ packet queue       |
                                                    | ├─ ordered buffer     |
                                                    | ├─ duplicate check    |
                                                    | └─ timeout/recovery   |
                                                    +-----------------------+
```

The code is organized into three packages:

| Package | Responsibility |
| --- | --- |
| `client` | Starts simulated clients and creates ordered, unordered, repeated, or intentionally missing packet sequences. |
| `server` | Receives all datagrams, routes them by source IP and port, and manages one worker thread per client. |
| `common` | Defines the wire-message model, status enum, sequence comparator, and packet logging helpers. |

### Message flow

1. A `ClientThread` serializes a `StructuredMessage` and sends it through a `DatagramSocket`.
2. `UDPServer` receives the datagram and finds or creates a `ClientHandlerThread` for that client endpoint.
3. The handler queues, parses, deduplicates, and sorts messages by sequence number.
4. The handler sends `OK` or `REPEATED_PACKET` immediately.
5. After five seconds, the handler checks whether its buffer has the expected packet count. If not, it sends `MISSING_PACKET`; otherwise, it logs the complete ordered sequence and ends.

## Requirements

- Java Development Kit (JDK) 8 or newer
- A free local UDP port `9876`

No external libraries or build tools are required.

## Repository layout

```text
src/
├── client/
│   ├── UDPClient.java            # Demo entry point; starts simulated clients
│   ├── ClientThread.java         # Sends packets and simulates test scenarios
│   └── TestCaseEnum.java         # Available scenarios
├── server/
│   ├── UDPServer.java            # Server entry point and packet dispatcher
│   └── ClientHandlerThread.java  # Per-client buffer and recovery logic
└── common/
    ├── StructuredMessage.java    # Wire-format serialization and parsing
    ├── EnumReturn.java           # Acknowledgement statuses
    ├── StructuredMessagesComparator.java
    └── LogUtils.java
```

## Run

From the repository root, compile the sources:

```bash
javac -d out $(find src -name '*.java')
```

In one terminal, start the server:

```bash
java -cp out server.UDPServer
```

In a second terminal, start the client demo:

```bash
java -cp out client.UDPClient
```

The console output shows each sent and received datagram, handler creation, acknowledgement, buffering, and the final ordered sequence. The server keeps running until it is stopped manually (for example, with `Ctrl+C`).

### Expected demo behavior

`UDPClient` starts one ordered client and one client that omits packet `5`.

- The ordered client receives `OK` acknowledgements for all ten packets.
- The lost-message client receives `OK` for the packets it sends, waits for the handler timeout, then receives `MISSING_PACKET` and retransmits packet `5`.
- Each handler prints its sorted message buffer once all ten packets are present. Because completion is checked on the timeout cycle, this message can appear up to five seconds after the final packet arrives.

## Troubleshooting

| Symptom | Likely cause and action |
| --- | --- |
| `java.net.BindException: Address already in use` | Another process is using UDP port `9876`. Stop that process or change `SERVER_PORT` in both `UDPServer` and `ClientThread`. |
| Client waits for a response | Start `UDPServer` before `UDPClient`, and confirm that both use the same host and port. |
| Output is very long or contains blank/null-looking characters | This is caused by the current fixed-size buffer logging. It does not prevent the demo from completing. |

## Configuration and extension points

- Change the server port in `UDPServer` and `ClientThread` together.
- Change the target host in `ClientThread` (`SERVER_IP`), which defaults to loopback.
- Select scenarios in `UDPClient` by changing the `TestCaseEnum` passed to each `ClientThread`.
- Change the sample payloads in `ClientThread.messages`.
- Adjust the handler timeout through `ClientHandlerThread.TIMEOUT`.

## Current limitations

The project intentionally keeps the protocol small so its behavior is easy to inspect. Important limitations include:

- The recovery signal identifies only that the batch is incomplete; it does not specify which sequence numbers are missing.
- Delivery is not guaranteed: clients wait indefinitely for acknowledgements and do not use retransmission timers, retry limits, checksums, or congestion control.
- A handler declares success based on message count, not on a contiguous sequence-number range.
- The global handler list is never cleaned up. Once a handler finishes, a later datagram from the same IP and source port can be routed to that stopped handler.
- The in-memory queues and handler registry have no synchronization or lifecycle management for sustained concurrent traffic.
- Messages use a simple colon-delimited text format without escaping, so payload data cannot safely contain `:`.
- Logging converts complete fixed-size UDP buffers to strings, which makes terminal output noisy with trailing null bytes.

These trade-offs are reasonable for a classroom demonstration, but they should be addressed before using this design in a real service.
