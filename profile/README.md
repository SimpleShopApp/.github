# SimpleShop - Sistema Distribuito Event-Driven con DDD

Proof of Technology di un sistema e-commerce distribuito che dimostra l'applicazione di Domain-Driven Design (DDD), architettura a microservizi, comunicazione asincrona tramite RabbitMQ e pattern Saga/Process Manager.

Ho scritto questa applicazione per studiare il DDD e pattern di resilienza e monitoraggio. Usala a tuo rischio.

Applicazione sviluppata con claude code e pensata come cookbook per lo sviluppo agentico enterprise.

## Architettura

```mermaid
flowchart TB
    Cliente([Cliente])

    subgraph BC["Bounded Contexts"]
        direction LR
        Sales["<b>Sales</b><br/>:8081<br/><i>Core Domain</i>"]
        Inventory["<b>Inventory</b><br/>:8082<br/><i>Supporting</i>"]
        Payment["<b>Payment</b><br/>:8083<br/><i>Generic</i>"]
        Shipping["<b>Shipping</b><br/>:8084<br/><i>Supporting</i>"]
        CS["<b>Customer Service</b><br/>:8085<br/><i>Process Manager</i>"]
    end

    subgraph Infra["Infrastruttura"]
        direction LR
        DB[("PostgreSQL<br/>5 schema isolati")]
        MQ["RabbitMQ<br/>Event Bus<br/>(Transactional Outbox)"]
    end

    Cliente -->|REST| Sales
    Cliente -->|REST| CS

    BC -.->|JDBC| DB
    BC <-->|AMQP| MQ

    classDef core fill:#1168bd,stroke:#0b4884,color:#fff
    classDef supporting fill:#438dd5,stroke:#2e6295,color:#fff
    classDef generic fill:#85bbf0,stroke:#5d9dd8,color:#000
    classDef infra fill:#666,stroke:#333,color:#fff
    classDef client fill:#08427b,stroke:#052e56,color:#fff

    class Sales core
    class Inventory,Shipping,CS supporting
    class Payment generic
    class DB,MQ infra
    class Cliente client
```

**Pattern utilizzati:**
- **Transactional Outbox** — Eventi salvati nella stessa transazione dell'aggregato, pubblicati da poller asincrono
- **Saga Coreografata** — Order Fulfillment orchestrato tramite eventi tra i Bounded Context
- **Process Manager** — Return/Refund gestito da ReturnRequest aggregate in Customer Service
- **Anti-Corruption Layer** — Payment isola il dominio dal gateway di pagamento esterno

### Order Fulfillment Saga

```mermaid
sequenceDiagram
    participant C as Cliente
    participant S as Sales
    participant I as Inventory
    participant P as Payment
    participant Sh as Shipping

    C->>S: POST /api/orders
    activate S
    S-->>S: Order (PENDING)
    S-)I: OrderCreated
    deactivate S

    activate I
    I-->>I: Riserva stock
    I-)P: InventoryReserved
    deactivate I

    activate P
    P-->>P: Autorizza pagamento
    P-)S: PaymentAuthorized
    deactivate P

    activate S
    S-->>S: Order (CONFIRMED)
    S-)Sh: OrderConfirmed
    deactivate S

    activate Sh
    Sh-->>Sh: Schedula spedizione
    Sh-)S: ShipmentDispatched
    deactivate Sh

    activate S
    S-->>S: Order (SHIPPED) ✓
    deactivate S

    Note over S,Sh: Tutti gli eventi transitano via Transactional Outbox → RabbitMQ
```

## Disclaimer

Non adottare microservizi per moda, ma solo quando i driver di business (scalabilità, deploy indipendente, team autonomi) lo giustificano. Il modular monolith è spesso un'alternativa valida che offre molti dei benefici di manutenibilità senza la complessità distribuita.

DDD introduce complessità architetturale significativa, richiede una curva di apprendimento ripida per tutto il team, collaborazione continua con domain experts e può risultare un over-engineering costoso. Se applicato a domini semplici o con team inesperti meglio partire con approcci più semplici ([Transaction Script](https://martinfowler.com/eaaCatalog/transactionScript.html) o [Active Record](https://en.wikipedia.org/wiki/Active_record_pattern)) e adottare DDD solo quando la complessità del business lo giustifica realmente.

Se vuoi comunque utilizzare DDD e microservizi event-driven, continua a leggere.

## Libri consigliati

- Inizia con [Domain-Driven Design](https://amzn.to/4qsLoZ4).
- Prosegui con [Software Architecture: The Hard Parts](https://amzn.to/3MtgfXw) per i pattern Saga e workflow distribuiti
- Approfondisci con [Enterprise Integration Patterns](https://amzn.to/4rMGR52) per i dettagli di messaging e Process Manager
- Completa con [Designing Data-Intensive Applications](https://amzn.to/4qsKiws) per la teoria dei sistemi distribuiti
- Consulta [Release It!](https://amzn.to/3MhDmo6) per i pattern di resilienza in produzione

# Sviluppo MVP

## Backend

## Frontend

## Test end to end

# Requisiti non funzionali

## Prestazioni e Scalabilità

Tempi di risposta, throughput e ottimizzazione delle performance, horizontal scaling, auto-scaling e load balancing

## Affidabilità

Resilienza, fault tolerance e disaster recovery

## Sicurezza

Principi di sicurezza, validazione e protezione dei dati

## Monitoraggio

Logging, metriche, alerting e osservabilità
