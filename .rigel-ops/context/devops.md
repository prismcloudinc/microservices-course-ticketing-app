---
repository: microservices-course-ticketing-app
platform: kubernetes
resource_count: 24
deployment_pattern: containers-k8s
last_analyzed: 2026-11-10T20:25:08.791Z
schema_version: "1.0"
---

# DevOps Infrastructure

## Summary

A microservices ticketing platform deployed on Kubernetes with six independent services (auth, tickets, orders, payments, client, and expiration) communicating via NATS Streaming for event-driven architecture. Each backend service has its own MongoDB instance for data isolation, with Redis supporting the expiration service for delayed job processing. An NGINX ingress controller routes external traffic to services through ticketing.dev domain.

**Platform**: kubernetes
**Deployment Pattern**: containers-k8s

## Resources

| Type | Name | Description |
|------|------|-------------|
| apps/v1/Deployment | auth-depl | Authentication service for user registration and login |
| v1/Service | auth-srv | Service exposing auth deployment on port 3000 |
| apps/v1/Deployment | auth-mongo-depl | MongoDB instance for auth service data storage |
| v1/Service | auth-mongo-srv | Service exposing MongoDB for auth on port 27017 |
| apps/v1/Deployment | tickets-depl | Tickets service for ticket management |
| v1/Service | tickets-srv | Service exposing tickets deployment on port 3000 |
| apps/v1/Deployment | tickets-mongo-depl | MongoDB instance for tickets service data storage |
| v1/Service | tickets-mongo-srv | Service exposing MongoDB for tickets on port 27017 |
| apps/v1/Deployment | orders-depl | Orders service for order processing and management |
| v1/Service | orders-srv | Service exposing orders deployment on port 3000 |
| apps/v1/Deployment | orders-mongo-depl | MongoDB instance for orders service data storage |
| v1/Service | orders-mongo-srv | Service exposing MongoDB for orders on port 27017 |
| apps/v1/Deployment | payments-depl | Payments service for processing Stripe payments |
| v1/Service | payments-srv | Service exposing payments deployment on port 3000 |
| apps/v1/Deployment | payments-mongo-depl | MongoDB instance for payments service data storage |
| v1/Service | payments-mongo-srv | Service exposing MongoDB for payments on port 27017 |
| apps/v1/Deployment | client-depl | Frontend client application served on port 3000 |
| v1/Service | client-srv | Service exposing client deployment on port 3000 |
| apps/v1/Deployment | expiration-depl | Expiration service for order timeout handling |
| apps/v1/Deployment | expiration-redis-depl | Redis cache for expiration service delayed job processing |
| v1/Service | expiration-redis-srv | Service exposing Redis for expiration service on port 6379 |
| apps/v1/Deployment | nats-depl | NATS Streaming server for event-driven communication between services |
| v1/Service | nats-srv | Service exposing NATS Streaming on client port 4222 and monitoring port 8222 |
| networking.k8s.io/v1/Ingress | ingress-service | NGINX ingress controller routing traffic to microservices |

## Networking

- **Ingress**: NGINX ingress at ticketing.dev routing to auth, tickets, orders, payments, and client services

## External Dependencies

| Type | Name | Details |
|------|------|---------|
| database | auth-mongo-srv | MongoDB for auth service at mongodb://auth-mongo-srv:27017/auth |
| database | tickets-mongo-srv | MongoDB for tickets service at mongodb://tickets-mongo-srv:27017/tickets |
| database | orders-mongo-srv | MongoDB for orders service at mongodb://orders-mongo-srv:27017/orders |
| database | payments-mongo-srv | MongoDB for payments service at mongodb://payments-mongo-srv:27017/payments |
| cache | expiration-redis-srv | Redis cache for expiration service at expiration-redis-srv:6379 |
| queue | nats-srv | NATS Streaming message broker at nats-srv:4222 with cluster ID ticketing for event-driven communication |
| service | jwt-secret | Kubernetes secret containing JWT_KEY for auth, orders, tickets, and payments services |
| service | stripe-secret | Kubernetes secret containing STRIPE_KEY for payments service |
| service | stripe-client-secret | Kubernetes secret containing STRIPE_CLIENT_KEY for client service |

## Environments

- **dev**: Single replica deployments, host set to ticketing.dev, liveness probes configured for health checks
