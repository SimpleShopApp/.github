# SimpleShop - Sistema Distribuito Event-Driven con DDD

Proof of Technology di un sistema e-commerce distribuito che dimostra l'applicazione di Domain-Driven Design (DDD), architettura a microservizi, comunicazione asincrona tramite RabbitMQ e pattern Saga/Process Manager.

## 🏗️ Architettura

### Bounded Contexts

```
  SALES (Core) ──eventi──> INVENTORY (Supporting)
       │                        │
       │                        v
       └──eventi──> PAYMENT (Generic) ──> SHIPPING (Supporting)
                                               │
                                               v
                                    CUSTOMER SERVICE (Supporting)
```

| Bounded Context  | Tipo       | Porta | Responsabilità                          |
|-----------------|------------|-------|-----------------------------------------|
| Sales           | Core       | 8081  | Gestione ordini e ciclo di vita        |
| Inventory       | Supporting | 8082  | Gestione magazzino e prenotazioni      |
| Payment         | Generic    | 8083  | Autorizzazioni e rimborsi pagamenti    |
| Shipping        | Supporting | 8084  | Gestione spedizioni e resi             |
| Customer Service| Supporting | 8085  | Gestione resi e process manager        |

### Stack Tecnologico

- **Java 21** (LTS)
- **Spring Boot 4.0.2** (ultima versione stabile)
- **PostgreSQL 16** (un'istanza, 5 schema separati)
- **RabbitMQ 3.13** (con management UI)
- **Maven** (progetto multi-modulo)
- **Flyway** (migrazioni database)
- **Testcontainers** (test integrazione)
- **JUnit 5** (test unitari)

## 🚀 Avvio Rapido

### Prerequisiti

- **Docker & Docker Compose** (Raccomandato) - Per esecuzione containerizzata
- **Java 21** (Opzionale) - Solo se esegui senza Docker
- **Maven 3.9+** (Opzionale) - Solo se esegui senza Docker

### Opzione A: Docker (Raccomandato) 🐳

**Avvio con un comando:**

```bash
cd simpleshop
./build-and-run.sh
```

Lo script automaticamente:
- ✅ Verifica Docker e porte disponibili
- ✅ Compila il progetto con Maven
- ✅ Build delle immagini Docker
- ✅ Avvia tutti i 7 servizi
- ✅ Mostra endpoint e comandi utili

**Stop servizi:**

```bash
./stop.sh

# Oppure stop con rimozione dati
docker compose down -v
```

**Monitoraggio:**

```bash
# Stato servizi
docker compose ps

# Logs tutti i servizi
docker compose logs -f

# Logs singolo servizio
docker compose logs -f sales
```

**Guida completa**: Vedi [DOCKER.md](DOCKER.md)

### Opzione B: Locale (Senza Docker)

**1. Avviare infrastruttura (solo PostgreSQL e RabbitMQ):**

```bash
docker compose up -d postgres rabbitmq
```

**2. Compilare il progetto:**

```bash
mvn clean install
```

**3. Avviare i servizi in terminali separati:**

```bash
# Terminal 1 - Sales
cd simpleshop-sales && mvn spring-boot:run

# Terminal 2 - Inventory
cd simpleshop-inventory && mvn spring-boot:run

# Terminal 3 - Payment
cd simpleshop-payment && mvn spring-boot:run

# Terminal 4 - Shipping
cd simpleshop-shipping && mvn spring-boot:run

# Terminal 5 - Customer Service
cd simpleshop-customerservice && mvn spring-boot:run
```

### 4. Testare il sistema

**Creare un ordine:**

```bash
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "c0000000-0000-0000-0000-000000000001",
    "items": [
      {
        "productId": "11111111-1111-1111-1111-111111111111",
        "quantity": 2,
        "unitPrice": 999.99,
        "currency": "USD"
      }
    ]
  }'
```

Risposta:
```json
{
  "orderId": "uuid-generato"
}
```

**Monitorare gli eventi:**

Accedi a RabbitMQ Management UI: http://localhost:15672
- Username: `simpleshop`
- Password: `simpleshop`

Naviga su **Exchanges** e osserva gli eventi che fluiscono tra i bounded contexts.

## 📐 Pattern Architetturali

### 1. Transactional Outbox

Ogni bounded context salva gli eventi in una tabella `outbox_events` nella stessa transazione dell'aggregato. Un poller asincrono (`OutboxEventPoller`) pubblica gli eventi su RabbitMQ, garantendo consegna **exactly-once**.

```
[Aggregato] ─┬─> [Tabella Aggregato]     }
             │                            } Stessa transazione
             └─> [Tabella outbox_events] }

[OutboxEventPoller] ─> legge outbox ─> pubblica su RabbitMQ
```

### 2. Saga Coreografata (Order Fulfillment)

La saga di elaborazione ordini è basata su **coreografia**: ogni servizio reagisce agli eventi e pubblica i propri.

```
Sales: OrderCreated
  ↓
Inventory: InventoryReserved
  ↓
Payment: PaymentAuthorized
  ↓
Sales: OrderConfirmed
  ↓
Shipping: ShipmentDispatched
  ↓
Sales: OrderShipped ✓
```

**Compensazione** in caso di fallimento:
- PaymentFailed → Sales pubblica OrderCancelled → Inventory rilascia stock

### 3. Process Manager (Return/Refund)

Il processo di reso è gestito da un **Process Manager** (aggregato `ReturnRequest` in Customer Service) con logica di branching complessa basata su ispezione item.

```
ReturnRequest (Process Manager)
  ├─> Ispezione OK → Rimborso FULL + Ripristino inventario
  ├─> Ispezione DAMAGED → Rimborso PARTIAL + NO ripristino
  └─> Ispezione WRONG_ITEM → Rimborso FULL + Ripristino
```

### 4. Idempotenza

Ogni consumer mantiene una tabella `processed_events` per evitare duplicati:

```sql
CREATE TABLE processed_events (
    event_id UUID PRIMARY KEY,
    event_type VARCHAR(255),
    processed_at TIMESTAMP
);
```

Prima di processare un evento, verifica se `event_id` è già presente.

### 5. Anti-Corruption Layer (ACL)

Il bounded context Payment usa un ACL (`FakePaymentGateway`) per isolare il dominio dal gateway esterno.

## 📊 Database

### Schema Separation

Un'unica istanza PostgreSQL con 5 schema separati:

- `sales` - Ordini e line items
- `inventory` - Prodotti e prenotazioni
- `payment` - Transazioni pagamento
- `shipping` - Spedizioni
- `customerservice` - Richieste reso

Ogni schema ha:
- Tabelle aggregati
- Tabella `outbox_events`
- Tabella `processed_events`

### Migrazioni

Gestite con Flyway. Le migrazioni si trovano in:
```
simpleshop-{bc}/src/main/resources/db/migration/
```

## 🔀 RabbitMQ Topology

### Exchanges (Topic)

- `sales-events`
- `inventory-events`
- `payment-events`
- `shipping-events`
- `customerservice-events`

### Routing Keys

Formato: `{bounded-context}.{aggregate}.{event}`

Esempi:
- `sales.order.created`
- `inventory.reservation.failed`
- `payment.transaction.authorized`
- `shipping.shipment.dispatched`

### Queue Bindings

Ogni consumer dichiara le proprie queue con binding specifici:

```java
@Bean
public Binding paymentAuthorizedBinding(TopicExchange paymentExchange) {
    return BindingBuilder
        .bind(salesPaymentAuthorizedQueue())
        .to(paymentExchange)
        .with("payment.transaction.authorized");
}
```

## 🧪 Testing

### Struttura Test

```
simpleshop-{bc}/
├── src/test/java/
│   ├── domain/        # Unit test aggregati
│   ├── application/   # Test application service
│   └── integration/   # Test con Testcontainers
```

### Eseguire i test

```bash
# Tutti i test
mvn test

# Solo un modulo
cd simpleshop-sales
mvn test

# Test E2E
cd simpleshop-e2e-tests
mvn verify
```

## 📚 Documentazione

- [Modello di Dominio](docs/domain-model.md) - Aggregati, VO, eventi per BC
- [Order Fulfillment Saga](docs/order-fulfillment-saga.md) - Flusso vendita + compensazioni
- [Return/Refund Process](docs/return-refund-process.md) - Process manager post-vendita
- [Event Catalog](docs/event-catalog.md) - Catalogo eventi con schemi JSON

## 🛠️ Struttura Progetto

```
simpleshop/
├── pom.xml                          # Parent POM
├── docker-compose.yml               # Infrastruttura
├── docs/                            # Documentazione
├── simpleshop-common/               # Shared Kernel
├── simpleshop-sales/                # BC Sales
├── simpleshop-inventory/            # BC Inventory
├── simpleshop-payment/              # BC Payment
├── simpleshop-shipping/             # BC Shipping
├── simpleshop-customerservice/      # BC Customer Service
└── simpleshop-e2e-tests/            # Test end-to-end
```

### Struttura Bounded Context

Ogni BC segue Hexagonal Architecture:

```
simpleshop-{bc}/
├── domain/
│   ├── model/          # Aggregati, entità, VO
│   ├── event/          # Eventi di dominio
│   └── command/        # Comandi (opzionale)
├── application/        # Application services, saga
├── port/               # Interfacce repository (domain)
├── infrastructure/
│   ├── persistence/    # JPA entities, adapter
│   ├── messaging/      # RabbitMQ listener, poller
│   └── web/            # REST controller
```

## 🔧 Configurazione

### Application Properties

Ogni servizio ha un `application.yml` in `src/main/resources/`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/simpleshop
    username: simpleshop
    password: simpleshop

  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        default_schema: {bc-name}

  flyway:
    schemas: {bc-name}

  rabbitmq:
    host: localhost
    port: 5672

server:
  port: 808{1-5}
```

## 🚨 Troubleshooting

### PostgreSQL non si avvia

```bash
docker compose down -v
docker compose up -d postgres
```

### RabbitMQ management UI non accessibile

Verifica porta 15672:
```bash
docker compose logs rabbitmq
```

### Eventi non vengono pubblicati

Verifica outbox poller nei log:
```bash
grep "Found.*unpublished events" simpleshop-sales/logs/app.log
```

### Test falliscono con Testcontainers

Assicurati che Docker sia in esecuzione:
```bash
docker info
```

## 📈 Monitoraggio

### Metriche Chiave

- **RabbitMQ Management UI**: http://localhost:15672
  - Visualizza rate messaggi, code, exchange

- **Logs**: Ogni servizio logga su console
  - Livello: INFO per default
  - DEBUG per Spring AMQP

### Health Checks

```bash
# Verifica servizi
curl http://localhost:8081/actuator/health  # Sales
curl http://localhost:8082/actuator/health  # Inventory
# ...
```

## 🎯 Scenari di Test

### Happy Path - Order Fulfillment

1. POST order → Sales crea Order (PENDING)
2. Inventory riserva stock → InventoryReserved
3. Payment autorizza → PaymentAuthorized
4. Sales conferma → OrderConfirmed
5. Shipping schedula → ShipmentScheduled
6. Shipping spedisce → ShipmentDispatched
7. Sales aggiorna → OrderShipped ✓

### Compensation - Payment Failure

1. POST order → OrderCreated
2. Inventory → InventoryReserved
3. Payment → **PaymentFailed** ❌
4. Sales → OrderCancelled
5. Inventory → rilascia stock (compensazione)
6. Order finale: FAILED

### Return Process - Item OK

1. POST return → ReturnRequested
2. Approvazione → ReturnApproved
3. Shipping → ReturnLabelIssued
4. Warehouse → ReturnShipmentReceived
5. Ispezione → ItemInspected (OK)
6. Payment → RefundProcessed (FULL)
7. Inventory → InventoryRestored
8. ReturnCompleted ✓

## 🔐 Sicurezza

> ⚠️ **Nota**: Questo è un progetto PoC. Non usare in produzione senza:
> - Autenticazione/Autorizzazione (OAuth2, JWT)
> - Encryption at rest/in transit (TLS)
> - Secret management (Vault, K8s secrets)
> - Rate limiting
> - Input validation robusta

## 📄 Licenza

Progetto educativo - uso libero per apprendimento e sperimentazione.

Claude Coded.
