## MER — Benchmarks (conceitual)

```mermaid
erDiagram
  BENCHMARK ||--o{ CONTROL : has
  CONTROL   ||--o{ STATE_CHANGE : records

  BENCHMARK {
    string id PK
    string name
  }

  CONTROL {
    string id PK
    string benchmark_id FK
    string name
    string description
  }

  STATE_CHANGE {
    string id PK
    string control_id FK
    datetime changed_at
    enum state "ok|alarm"
    string note
  }

