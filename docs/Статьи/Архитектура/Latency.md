---
share: true
---

## Таблица latency с масштабом

| Event                               | Latency    | Scaled        |
|-------------------------------------|------------|---------------|
| 1 CPU cycle                         | 0.3 ns     | 1 s           |
| Level 1 cache access                | 0.9 ns     | 3 s           |
| Level 2 cache access                | 2.8 ns     | 9 s           |
| Main memory access                  | 120 ns     | 6 min         |
| SSD random read                     | 16 000 ns  | 15 h          |
| Network req/resp in same datacenter | 500 000 ns | 19 days       |
| Network req/resp internet SF to NY  | 40 ms      | 4 years       |
| TCP packet retransmit               | 1-3 s      | 105-317 years |
