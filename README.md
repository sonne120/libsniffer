
# WareHound Sniffer

A high-performance network packet sniffer and analyzer, designed for flexible capture, protocol analysis, and real-time statistics. Suitable for research, diagnostics, and network monitoring.


## Architecture

```
          ┌──────────────┐
          │ PcapSource   │  1 capture thread (pinned)
          │ (libpcap/    │
          │  Npcap)      │
          └──────┬───────┘
                 │ memcpy into BufferPool
                 ▼
        ┌────────────────────┐
        │ SpmcRingBuffer     │  capture ring
        │  <PacketView>      │  (single producer)
        └─────────┬──────────┘
                  │ work-stealing claim
       ┌──────────┼──────────┬───────────┐
       ▼          ▼          ▼           ▼
   ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
   │Mapper0│ │Mapper1│ │Mapper2│ │Mapper3│   N parser threads (pinned)
   └───┬───┘ └───┬───┘ └───┬───┘ └───┬───┘
       │ parse + flow hash └───┬─────┘
       └───────────────────────▼
              ┌────── Shuffler ───────┐
              │  hash(5-tuple) % R    │
              └──┬──────────┬─────────┘
                 │          │
                 ▼          ▼
        ┌──────────┐  ┌──────────┐
        │ SPSC[0]  │  │ SPSC[1]  │  …  per-reducer queues
        └────┬─────┘  └────┬─────┘
             ▼             ▼
        ┌────────┐    ┌────────┐
        │Reducer0│    │Reducer1│       R stat threads (pinned)
        └────┬───┘    └────┬───┘
             └──── release buffer back to BufferPool
```

## Build

### Linux (Debian/Ubuntu)

```bash
sudo apt install libpcap-dev cmake build-essential
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
sudo setcap cap_net_raw,cap_net_admin=eip build/simple_sniff   # optional
build/simple_sniff eth0 "tcp port 443"
```

### macOS

```bash
brew install libpcap cmake
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
sudo build/simple_sniff en0
```

### Windows (Npcap)

1. Install [Npcap](https://npcap.com/) (with WinPcap-compatible mode).
2. Download the [Npcap SDK](https://npcap.com/#download) and unzip somewhere,
   e.g. `C:\npcap-sdk`.
3. Build:

```cmd
cmake -S . -B build -DNPCAP_SDK=C:\npcap-sdk -A x64
cmake --build build --config Release
build\Release\simple_sniff.exe "\Device\NPF_{GUID}"
```

(Find the interface name with `getmac /v` or use `pcap_findalldevs`.)


## Suggested next steps

1. Swap own SPMC for **Disruptor-cpp** WorkerPool — better wait strategies,
   dependent-consumer barriers.
2. Replace `BufferPool`'s mutex with a Treiber stack (or per-thread caches).
3. Add an `AfPacketSource` for Linux (PACKET_MMAP / TPACKET_V3 = 10x libpcap).
4. Implement hazard pointers and a sharded flow table (one shard per reducer
   so no synchronization is needed on lookups).
5. Add a real benchmark harness — replay a pcap file in a tight loop and
   measure packets/sec at each stage.
6. ThreadSanitizer in CI — lock-free code without TSan is gambling.
