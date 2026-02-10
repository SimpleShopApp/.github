# SimpleShop - Sistema Distribuito Event-Driven con DDD

Proof of Technology di un sistema e-commerce distribuito che dimostra l'applicazione di Domain-Driven Design (DDD), architettura a microservizi, comunicazione asincrona tramite RabbitMQ e pattern Saga/Process Manager.

Ho scritto questa applicazione per studiare il DDD e pattern di resilienza e monitoraggio. Usala a tuo rischio.

Claude Coded.

## Disclaimer

Non adottare microservizi per moda, ma solo quando i driver di business (scalabilità, deploy indipendente, team autonomi) lo giustificano. Il modular monolith è spesso un'alternativa valida che offre molti dei benefici di manutenibilità senza la complessità distribuita.

DDD introduce complessità architetturale significativa, richiede una curva di apprendimento ripida per tutto il team, collaborazione continua con domain experts e può risultare un over-engineering costoso. Se applicato a domini semplici o con team inesperti meglio partire con approcci più semplici (Transaction Script o Active Record) e adottare DDD solo quando la complessità del business lo giustifica realmente.

## Libri consigliati

- Inizia con [Domain-Driven Design](https://amzn.to/4qsLoZ4).
- Prosegui con [Software Architecture: The Hard Parts](https://amzn.to/3MtgfXw) per i pattern Saga e workflow distribuiti
- Approfondisci con [Enterprise Integration Patterns](https://amzn.to/4rMGR52) per i dettagli di messaging e Process Manager
- Completa con [Designing Data-Intensive Applications](https://amzn.to/4qsKiws) per la teoria dei sistemi distribuiti
- Consulta [Release It!](https://amzn.to/3MhDmo6) per i pattern di resilienza in produzione

## Analisi

## Sviluppo

## Refactor: Downgrade java

## Deploy: Gitlab CI/CD su k3s

## Refactor: RabbitMQ to Kafka
