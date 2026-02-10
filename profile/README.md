# SimpleShop - Sistema Distribuito Event-Driven con DDD

Proof of Technology di un sistema e-commerce distribuito che dimostra l'applicazione di Domain-Driven Design (DDD), architettura a microservizi, comunicazione asincrona tramite RabbitMQ e pattern Saga/Process Manager.

Claude Coded.

## Disclaimer

Non adottare microservizi per moda, ma solo quando i driver di business (scalabilità, deploy indipendente, team autonomi) lo giustificano. Il modular monolith è spesso un'alternativa valida che offre molti dei benefici di manutenibilità senza la complessità distribuita. [Approfondisci Qui](https://amzn.to/4qrZ8mY).

DDD introduce complessità architetturale significativa, richiede una curva di apprendimento ripida per tutto il team, collaborazione continua con domain experts e può risultare un over-engineering costoso. Se applicato a domini semplici o con team inesperti meglio partire con approcci più semplici (Transaction Script o Active Record) e adottare DDD solo quando la complessità del business lo giustifica realmente. [Approfondisci qui](https://amzn.to/45Ykml6).

## Analisi

## Sviluppo

## Refactor: Downgrade java

## Deploy: Gitlab CI/CD su k3s

## Refactor: RabbitMQ to Kafka
