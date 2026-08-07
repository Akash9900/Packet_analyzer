# DPI Engine

Deep packet inspection for PCAP captures. Parses Ethernet/IP/TCP/UDP, extracts TLS SNI (and HTTP Host), classifies traffic by application, and optionally drops flows by IP, app, or domain.

## Layout

```
include/          Headers (pcap reader, parser, SNI extractor, types)
src/
  dpi_mt.cpp          Multi-threaded engine (default)
  main_working.cpp    Single-threaded engine
  pcap_reader.cpp     PCAP I/O
  packet_parser.cpp   Protocol parsing
  sni_extractor.cpp   TLS/HTTP inspection
  types.cpp           App classification helpers
generate_test_pcap.py Sample traffic generator
test_dpi.pcap         Sample capture
```

## Build

Requires a C++17 compiler. No external libraries.

**Multi-threaded (default):**

```bash
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine \
    src/dpi_mt.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

**Single-threaded:**

```bash
g++ -std=c++17 -O2 -I include -o dpi_simple \
    src/main_working.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

Or with CMake:

```bash
cmake -B build && cmake --build build
```

**Windows (MinGW):** same `g++` command; drop `-pthread` if unused, and use `-o dpi_engine.exe`.

## Run

```bash
./dpi_engine test_dpi.pcap output.pcap

./dpi_engine test_dpi.pcap output.pcap \
    --block-app YouTube \
    --block-ip 192.168.1.50 \
    --block-domain facebook

./dpi_engine input.pcap output.pcap --lbs 4 --fps 4
```

Regenerate the sample capture:

```bash
python3 generate_test_pcap.py
```
