# Deep Packet Inspection Engine

A C++17 packet-analysis engine that reads PCAP captures, parses network
protocols, extracts TLS Server Name Indication (SNI) and HTTP Host values,
classifies application traffic, applies flow-level blocking rules, and writes
the permitted traffic to a new PCAP file.

The repository includes a straightforward single-threaded implementation for
learning and a configurable multi-threaded implementation for larger captures.

## Verified sample results

Using the included `test_dpi.pcap` fixture, the engine:

- Processed **77 packets** and **5,738 bytes**
- Parsed **73 TCP** and **4 UDP** packets
- Extracted **18 domain/SNI values**
- Observed traffic across **18 protocol and application categories**
- Supports **23 total protocol/application classifications**
- Dropped **2 of 2 targeted test flows** when YouTube and TikTok blocking was enabled
- Distributed work across **2 load-balancer threads** and **4 fast-path workers** in the default test configuration

These values describe the included deterministic sample capture; they are not
production throughput benchmarks.

## Features

- Ethernet, IPv4, TCP, and UDP packet parsing
- TLS Client Hello SNI extraction
- HTTP Host header extraction
- Stateful five-tuple flow tracking
- Application classification for services such as Google, YouTube, Facebook,
  Netflix, Microsoft, TikTok, Spotify, GitHub, and others
- Blocking by source IP, application, or domain
- Single-threaded and multi-threaded processing modes
- Configurable load-balancer and fast-path worker counts
- Filtered PCAP output and a terminal processing report
- No external runtime libraries

## Processing pipeline

```text
Input PCAP
    |
    v
Packet reader
    |
    v
Ethernet/IP/TCP/UDP parser
    |
    v
TLS SNI or HTTP Host extraction
    |
    v
Five-tuple flow tracking and application classification
    |
    +---- blocked flow ----> dropped
    |
    +---- allowed flow ----> output PCAP
```

The multi-threaded engine uses a two-stage topology:

```text
Reader -> Load-balancer threads -> Fast-path workers -> Writer/report
```

Packets from the same flow are consistently assigned to the same worker so
that connection state and blocking decisions remain coherent.

## Repository structure

```text
include/
  packet_parser.h       Protocol parser declarations
  pcap_reader.h         PCAP reader and writer declarations
  platform.h            Portable byte-order helpers
  sni_extractor.h       TLS SNI and HTTP Host extraction
  types.h               Flow, packet, statistics, and application types
src/
  dpi_mt.cpp            Multi-threaded engine
  main_working.cpp      Single-threaded engine
  packet_parser.cpp     Ethernet/IP/TCP/UDP parsing
  pcap_reader.cpp       PCAP input and output
  sni_extractor.cpp     Application-layer inspection
  types.cpp             Flow formatting and app classification
CMakeLists.txt          CMake build configuration
generate_test_pcap.py   Deterministic test-capture generator
test_dpi.pcap           Sample capture
```

## Requirements

- macOS, Linux, or Windows
- A C++17 compiler such as Clang or GCC
- CMake 3.16 or newer for the CMake build
- Python 3 only when regenerating the sample capture

## Build

### CMake

```bash
cmake -S . -B build
cmake --build build
```

This produces:

- `build/dpi_engine` — multi-threaded engine
- `build/dpi_simple` — single-threaded engine

### Direct compiler command

Multi-threaded:

```bash
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine \
    src/dpi_mt.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

Single-threaded:

```bash
g++ -std=c++17 -O2 -I include -o dpi_simple \
    src/main_working.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

On macOS, `clang++` can be used in place of `g++`.

## Usage

Analyze the included capture:

```bash
./build/dpi_engine test_dpi.pcap output.pcap
```

Apply blocking rules:

```bash
./build/dpi_engine test_dpi.pcap blocked_output.pcap \
    --block-app YouTube \
    --block-app TikTok \
    --block-ip 192.168.1.50 \
    --block-domain facebook
```

Configure multi-threading:

```bash
./build/dpi_engine input.pcap output.pcap --lbs 4 --fps 4
```

This creates four load-balancer threads with four fast-path workers assigned to
each load balancer.

Regenerate the sample capture:

```bash
python3 generate_test_pcap.py
```

## How classification works

For HTTPS traffic, the engine inspects the TLS Client Hello and extracts the
plaintext SNI hostname before application data becomes encrypted. Known domain
patterns are mapped to application categories. HTTP traffic is classified
using its Host header, while DNS is identified from its transport ports.

Blocking decisions are stored on the flow. Once a flow matches a configured
rule, subsequent packets belonging to that flow are dropped.

## Scope and limitations

- The engine processes offline PCAP files; it does not capture live interfaces.
- It currently focuses on IPv4 traffic.
- TLS classification depends on the relevant Client Hello being present.
- Encrypted Client Hello can prevent SNI-based classification.
- QUIC/HTTP3 inspection is not implemented.
- The included metrics validate behavior, not maximum throughput.

## Future improvements

- Add live packet capture
- Add IPv6 and QUIC/HTTP3 inspection
- Reassemble TLS handshakes split across multiple packets
- Load signatures and blocking rules from configuration files
- Add automated tests and performance benchmarks
- Export structured JSON or Prometheus-compatible statistics
