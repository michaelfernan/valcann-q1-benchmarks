# DESAFIO: PRIMEIRA QUESTÃO  
**Banco de Dados (conceitual + índices básicos)**  

**Título:** Benchmarks — estado atual e histórico  

---

## Mini-mundo
- **Benchmark** `(id, name)` tem vários **Controles`.  
- **Controle** `(id, name, description)` pertence a um **Benchmark** e possui um **estado** (`ok | alarm`).  
- Deve ser possível registrar **mudanças de estado** para reconstruir o histórico.  

---

## Cenários
- **(Q1)** Listar Benchmark com seus Controles e o estado atual.  
- **(Q2)** Listar Benchmark com seus Controles e as mudanças de estado em um intervalo.  
- **(Q3)** Obter Benchmark com seus Controles e o estado em uma data/hora X.  

---

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

## Regras

- O **estado atual** de um controle é o último registro em `STATE_CHANGE` (maior `changed_at`).  
- O **histórico** é obtido pela sequência de registros em `STATE_CHANGE` por `control_id`, ordenada por `changed_at` (*append-only*).  
- `changed_at` deve ser armazenado em **UTC**.  

---

## Índices básicos (e por quê)

- **CONTROL(benchmark_id)**  
  Para consultar controles de um benchmark sem *table scan*.  

- **STATE_CHANGE(control_id, changed_at)**  
  Índice principal para:  
  - recuperar o último estado de cada controle;  
  - percorrer histórico ordenado;  
  - aplicar filtros por intervalo de tempo.  

### Opcionais
- `STATE_CHANGE(changed_at)` → buscas globais por janelas de tempo.  
- Índice parcial `STATE_CHANGE WHERE state='alarm'` → útil se alarmes forem raros.  
- *Snapshot* opcional: tabela derivada `ControlCurrentState(control_id, state, changed_at)` com `UNIQUE(control_id)` para consultas rápidas.  

---

## Consultas de exemplo (PostgreSQL-like, **sem DDL**)

### (Q1) Benchmark + Controles + estado atual
```sql
SELECT b.id AS benchmark_id, b.name AS benchmark_name,
       c.id AS control_id, c.name AS control_name, c.description,
       sc.state AS current_state, sc.changed_at AS current_changed_at
FROM benchmark b
JOIN control c ON c.benchmark_id = b.id
LEFT JOIN LATERAL (
  SELECT state, changed_at
  FROM state_change
  WHERE control_id = c.id
  ORDER BY changed_at DESC
  LIMIT 1
) sc ON TRUE
WHERE b.id = :benchmark_id;


### (Q2) Benchmark + Controles + mudanças de estado em um intervalo
```sql
SELECT b.id, b.name,
       c.id AS control_id, c.name AS control_name,
       sc.state, sc.changed_at
FROM benchmark b
JOIN control c ON c.benchmark_id = b.id
JOIN state_change sc ON sc.control_id = c.id
WHERE b.id = :benchmark_id
  AND sc.changed_at >= :from_ts
  AND sc.changed_at <  :to_ts
ORDER BY c.id, sc.changed_at;


### (Q3) Benchmark + Controles + estado em uma data/hora X
```sql
SELECT b.id, b.name,
       c.id AS control_id, c.name AS control_name, c.description,
       scx.state, scx.changed_at
FROM benchmark b
JOIN control c ON c.benchmark_id = b.id
LEFT JOIN LATERAL (
  SELECT state, changed_at
  FROM state_change
  WHERE control_id = c.id
    AND changed_at <= :at_ts
  ORDER BY changed_at DESC
  LIMIT 1
) scx ON TRUE
WHERE b.id = :benchmark_id;

