# TicketHub — Plataforma de Venda de Ingressos em Microservices

Projeto de estudo para praticar **sistemas distribuídos**, **banco de dados isolado por serviço**, **comunicação síncrona e assíncrona**, **consistência eventual** e **padrões de resiliência**.

A ideia é um "Sympla / Ticketmaster simplificado": organizadores cadastram eventos, clientes compram ingressos para setores/assentos, pagam e recebem o ingresso com QR code.

---

## Sumário

1. [Visão geral da arquitetura](#1-visão-geral-da-arquitetura)
2. [Regras gerais (valem para todos os serviços)](#2-regras-gerais-valem-para-todos-os-serviços)
3. [Serviços](#3-serviços)
   - [3.1 API Gateway](#31-api-gateway)
   - [3.2 Identity Service](#32-identity-service)
   - [3.3 Catalog Service](#33-catalog-service)
   - [3.4 Inventory Service](#34-inventory-service)
   - [3.5 Orders Service](#35-orders-service)
   - [3.6 Payments Service](#36-payments-service)
   - [3.7 Tickets Service](#37-tickets-service)
   - [3.8 Notifications Service](#38-notifications-service)
   - [3.9 Search Service](#39-search-service)
4. [Catálogo de eventos (mensageria)](#4-catálogo-de-eventos-mensageria)
5. [Fluxo principal: compra de ingresso (Saga)](#5-fluxo-principal-compra-de-ingresso-saga)
6. [Padrões obrigatórios](#6-padrões-obrigatórios)
7. [Infraestrutura](#7-infraestrutura)
8. [Roteiro de desenvolvimento](#8-roteiro-de-desenvolvimento)
9. [Cenários de teste para validar o aprendizado](#9-cenários-de-teste-para-validar-o-aprendizado)

---

## 1. Visão geral da arquitetura

```
                         ┌──────────────┐
        Cliente  ───────▶│ API Gateway  │
                         └──────┬───────┘
                                │ REST (síncrono)
   ┌──────────┬──────────┬──────┴────┬──────────┬──────────┐
   ▼          ▼          ▼           ▼          ▼          ▼
Identity   Catalog   Inventory    Orders    Tickets    Search
(Postgres) (Mongo)   (Postgres    (Postgres) (Mongo)   (Elastic)
                      + Redis)       │
                         ▲    gRPC   │
                         └───────────┤ (Orders → Catalog / Inventory)
                                     │
            ════════════════ Message Broker (RabbitMQ / Kafka) ════════════════
                 ▲            ▲             ▲             ▲             ▲
              Catalog     Inventory      Orders       Payments     Notifications
                                                     (Postgres)    (sem banco)
```

| Serviço | Responsabilidade principal | Banco | Síncrono | Assíncrono |
|---|---|---|---|---|
| API Gateway | Porta de entrada, autenticação, roteamento | — | ✅ | — |
| Identity | Usuários, login, JWT | PostgreSQL | ✅ | publica |
| Catalog | Eventos, setores, preços | MongoDB | ✅ | publica |
| Inventory | Assentos, reservas, concorrência | PostgreSQL + Redis | ✅ | publica / consome |
| Orders | Pedidos e orquestração da Saga | PostgreSQL | ✅ | publica / consome |
| Payments | Cobrança (gateway simulado) | PostgreSQL | ✅ (webhook) | publica / consome |
| Tickets | Emissão e validação de ingressos | MongoDB | ✅ | publica / consome |
| Notifications | E-mails / push simulados | — (ou Redis) | — | consome |
| Search | Busca de eventos (read model) | Elasticsearch | ✅ | consome |

---

## 2. Regras gerais (valem para todos os serviços)

1. **Banco isolado:** cada serviço tem o seu próprio banco. **Nenhum serviço acessa o banco de outro.** Se precisar de um dado, pede via API ou mantém uma cópia local alimentada por eventos.
2. **Sem chave estrangeira entre serviços:** referências a outros serviços são apenas IDs (ex.: `userId` no Orders é só um UUID, não uma FK).
3. **IDs são UUID** gerados pelo próprio serviço.
4. **Health check:** todo serviço expõe `GET /health` (liveness) e `GET /ready` (readiness — banco e broker conectados).
5. **Correlation ID:** todo request/evento carrega um `X-Correlation-Id`, repassado em chamadas síncronas e no header das mensagens.
6. **Logs estruturados** (JSON) contendo `service`, `correlationId`, `level`, `message`.
7. **Tracing** com OpenTelemetry em todas as chamadas HTTP/gRPC e publicações/consumos de mensagens.
8. **Configuração por variável de ambiente** (nada de credencial hardcoded).
9. **Cada serviço sobe sozinho** com seu `Dockerfile`; o `docker-compose` na raiz sobe o ecossistema inteiro.
10. **Contratos versionados:** eventos têm campo `version`; endpoints ficam em `/v1/...`.

---

## 3. Serviços

### 3.1 API Gateway

**Responsabilidade:** ser o único ponto de entrada para o cliente.

**Obrigações**
- Rotear requisições para os serviços (`/v1/auth/*` → Identity, `/v1/events/*` → Catalog, etc.).
- Validar o JWT (assinatura e expiração) e repassar `userId` e `role` nos headers internos (`X-User-Id`, `X-User-Role`).
- Gerar o `X-Correlation-Id` quando o cliente não enviar.
- Rate limiting por IP/usuário (ex.: 100 req/min).
- Não conter regra de negócio.

**Pode ser:** Kong, Traefik, NGINX, YARP, Spring Cloud Gateway ou um gateway próprio simples.

---

### 3.2 Identity Service

**Responsabilidade:** identidade e autenticação dos usuários.

**Banco:** PostgreSQL

**Modelo de dados**
```
users
  id            UUID PK
  name          VARCHAR
  email         VARCHAR UNIQUE
  password_hash VARCHAR
  role          ENUM('CUSTOMER', 'ORGANIZER', 'ADMIN')
  created_at    TIMESTAMP
```

**Endpoints (síncrono)**
| Método | Rota | Descrição |
|---|---|---|
| POST | `/v1/auth/register` | Cadastra usuário |
| POST | `/v1/auth/login` | Retorna access token (JWT) e refresh token |
| POST | `/v1/auth/refresh` | Renova o access token |
| GET | `/v1/users/me` | Dados do usuário logado |
| GET | `/v1/users/{id}` | (interno) dados básicos de um usuário |

**Eventos publicados:** `UserRegistered`

**Obrigações / regras**
- Senha armazenada com hash forte (bcrypt/argon2).
- E-mail único.
- JWT com expiração curta (ex.: 15 min) + refresh token.
- Expor a chave pública (JWKS) ou segredo compartilhado para o Gateway validar tokens.

---

### 3.3 Catalog Service

**Responsabilidade:** cadastro e gestão dos eventos (shows, peças, jogos) e seus setores/preços.

**Banco:** MongoDB (um evento é um documento com setores embutidos)

**Modelo de dados**
```json
{
  "_id": "uuid",
  "organizerId": "uuid",
  "title": "Show da Banda X",
  "description": "...",
  "category": "MUSIC",
  "venue": { "name": "Arena Y", "city": "São Paulo", "address": "..." },
  "startsAt": "2026-12-10T21:00:00Z",
  "status": "DRAFT | PUBLISHED | CANCELED | FINISHED",
  "sectors": [
    { "id": "uuid", "name": "Pista", "price": 150.00, "capacity": 500, "numbered": false },
    { "id": "uuid", "name": "Cadeira A", "price": 300.00, "capacity": 100, "numbered": true }
  ]
}
```

**Endpoints (síncrono)**
| Método | Rota | Quem | Descrição |
|---|---|---|---|
| POST | `/v1/events` | ORGANIZER | Cria evento (DRAFT) |
| PUT | `/v1/events/{id}` | ORGANIZER dono | Edita evento |
| POST | `/v1/events/{id}/publish` | ORGANIZER dono | Publica evento |
| POST | `/v1/events/{id}/cancel` | ORGANIZER dono | Cancela evento |
| GET | `/v1/events/{id}` | público | Detalhe do evento |
| gRPC | `GetSectorPrice(eventId, sectorId)` | interno (Orders) | Preço atual e status |

**Eventos publicados:** `EventCreated`, `EventUpdated`, `EventPublished`, `EventCanceled`

**Obrigações / regras**
- Só o organizador dono pode editar/publicar/cancelar.
- Evento só pode ser publicado se tiver pelo menos um setor e data futura.
- Não é possível alterar capacidade de setor depois de publicado (só aumentar, opcional).
- **Não controla assentos disponíveis** — isso é responsabilidade do Inventory.

---

### 3.4 Inventory Service

**Responsabilidade:** controlar a disponibilidade de assentos/vagas e as reservas temporárias. É o serviço mais crítico em **concorrência**.

**Banco:** PostgreSQL (fonte da verdade) + Redis (locks e TTL de reservas)

**Modelo de dados**
```
seats
  id          UUID PK
  event_id    UUID
  sector_id   UUID
  label       VARCHAR        -- ex.: "A-12" (ou null para setor não numerado)
  status      ENUM('AVAILABLE', 'RESERVED', 'SOLD')
  version     INT            -- para lock otimista

reservations
  id          UUID PK
  order_id    UUID UNIQUE
  seat_ids    UUID[]
  status      ENUM('ACTIVE', 'CONFIRMED', 'RELEASED', 'EXPIRED')
  expires_at  TIMESTAMP
```

**Endpoints (síncrono)**
| Método | Rota | Descrição |
|---|---|---|
| GET | `/v1/events/{eventId}/availability` | Mapa de assentos/quantidade disponível por setor |
| gRPC/REST | `ReserveSeats(orderId, eventId, sectorId, quantity \| seatIds)` | Reserva por X minutos (chamado pelo Orders) |

**Eventos consumidos**
| Evento | Ação |
|---|---|
| `EventPublished` | Gera os assentos/vagas de cada setor |
| `EventCanceled` | Bloqueia todas as vendas do evento |
| `PaymentApproved` / `OrderConfirmed` | Reserva → `SOLD` |
| `OrderCanceled` | Libera os assentos → `AVAILABLE` |

**Eventos publicados:** `SeatsReserved`, `SeatsConfirmed`, `SeatsReleased`, `ReservationExpired`

**Obrigações / regras**
- **Nunca vender o mesmo assento duas vezes** (usar lock otimista com `version` ou `SELECT ... FOR UPDATE`, ou lock distribuído no Redis).
- Reserva expira em **10 minutos**; um job periódico (ou key expiration do Redis) libera assentos e publica `ReservationExpired`.
- `ReserveSeats` precisa ser **idempotente** por `orderId` (chamado duas vezes = mesma reserva).
- Limite de ingressos por usuário por evento (ex.: 4).

---

### 3.5 Orders Service

**Responsabilidade:** ciclo de vida do pedido e **orquestração da Saga** de compra.

**Banco:** PostgreSQL

**Modelo de dados**
```
orders
  id             UUID PK
  user_id        UUID
  event_id       UUID
  sector_id      UUID
  quantity       INT
  unit_price     DECIMAL      -- preço "congelado" no momento da compra
  total_amount   DECIMAL
  status         ENUM('PENDING', 'AWAITING_PAYMENT', 'CONFIRMED', 'CANCELED', 'EXPIRED')
  cancel_reason  VARCHAR NULL
  created_at     TIMESTAMP
  updated_at     TIMESTAMP

outbox
  id            UUID PK
  aggregate_id  UUID
  event_type    VARCHAR
  payload       JSONB
  created_at    TIMESTAMP
  published_at  TIMESTAMP NULL
```

**Endpoints (síncrono)**
| Método | Rota | Descrição |
|---|---|---|
| POST | `/v1/orders` | Cria pedido (header `Idempotency-Key` obrigatório) |
| GET | `/v1/orders/{id}` | Status do pedido |
| GET | `/v1/orders` | Pedidos do usuário logado |
| POST | `/v1/orders/{id}/cancel` | Cliente desiste antes de pagar |

**Chamadas síncronas que faz**
- **Catalog** (gRPC) → busca preço atual e confirma que o evento está `PUBLISHED`.
- **Inventory** (gRPC/REST) → reserva os assentos. Se falhar, o pedido nem é criado (retorna 409).

**Eventos publicados:** `OrderCreated`, `OrderConfirmed`, `OrderCanceled`

**Eventos consumidos**
| Evento | Ação |
|---|---|
| `PaymentApproved` | Pedido → `CONFIRMED`, publica `OrderConfirmed` |
| `PaymentFailed` | Pedido → `CANCELED`, publica `OrderCanceled` (compensação) |
| `ReservationExpired` | Pedido → `EXPIRED` |
| `EventCanceled` | Cancela pedidos pendentes do evento |

**Obrigações / regras**
- Máquina de estados explícita: transições inválidas são rejeitadas (ex.: `CANCELED` → `CONFIRMED`).
- Preço é congelado na criação do pedido.
- Usa **Outbox Pattern**: grava o pedido e o evento na mesma transação; um publisher envia ao broker.
- Usar **circuit breaker + timeout** nas chamadas ao Catalog e Inventory.

---

### 3.6 Payments Service

**Responsabilidade:** processar pagamentos através de um **gateway simulado**.

**Banco:** PostgreSQL

**Modelo de dados**
```
payments
  id               UUID PK
  order_id         UUID UNIQUE
  amount           DECIMAL
  method           ENUM('CREDIT_CARD', 'PIX')
  status           ENUM('PENDING', 'APPROVED', 'FAILED', 'REFUNDED')
  external_id      VARCHAR      -- id no "gateway fake"
  failure_reason   VARCHAR NULL
  created_at       TIMESTAMP

processed_messages
  message_id   UUID PK          -- para idempotência de consumo
  processed_at TIMESTAMP
```

**Endpoints (síncrono)**
| Método | Rota | Descrição |
|---|---|---|
| POST | `/v1/payments/webhook` | Callback do gateway fake (pode chegar duplicado!) |
| GET | `/v1/payments/order/{orderId}` | Status do pagamento |

**Eventos consumidos:** `OrderCreated` (inicia cobrança), `EventCanceled` (reembolsa pagamentos aprovados)

**Eventos publicados:** `PaymentApproved`, `PaymentFailed`, `PaymentRefunded`

**Gateway fake (obrigatório simular):**
- Latência aleatória (1–5s).
- ~70% aprovado, ~20% recusado, ~10% timeout/erro.
- Resposta via webhook, às vezes **enviado duas vezes**.

**Obrigações / regras**
- **Um pagamento por pedido** (`order_id UNIQUE`).
- Webhook e consumo de mensagens **idempotentes**.
- Retry com backoff exponencial em falhas temporárias do gateway; após N tentativas → `PaymentFailed`.

---

### 3.7 Tickets Service

**Responsabilidade:** emitir os ingressos após o pedido confirmado e validá-los na entrada do evento.

**Banco:** MongoDB

**Modelo de dados**
```json
{
  "_id": "uuid",
  "orderId": "uuid",
  "userId": "uuid",
  "eventId": "uuid",
  "seatLabel": "A-12",
  "code": "hash-unico-para-qr",
  "status": "VALID | USED | CANCELED",
  "issuedAt": "...",
  "usedAt": null
}
```

**Endpoints (síncrono)**
| Método | Rota | Descrição |
|---|---|---|
| GET | `/v1/tickets` | Ingressos do usuário logado |
| GET | `/v1/tickets/{id}/qrcode` | Imagem do QR code |
| POST | `/v1/tickets/validate` | Valida o código na portaria (ORGANIZER) |

**Eventos consumidos:** `OrderConfirmed` (emite), `EventCanceled` / `PaymentRefunded` (cancela)

**Eventos publicados:** `TicketsIssued`, `TicketUsed`

**Obrigações / regras**
- Um ingresso por assento/unidade do pedido.
- Emissão **idempotente**: receber `OrderConfirmed` duas vezes não gera ingressos duplicados.
- Ingresso só pode ser usado **uma vez**.

---

### 3.8 Notifications Service

**Responsabilidade:** avisar o usuário sobre o andamento da compra. **Apenas consome eventos.**

**Banco:** nenhum (opcional: Redis para deduplicação)

**Eventos consumidos**
| Evento | Notificação |
|---|---|
| `UserRegistered` | Boas-vindas |
| `OrderCreated` | "Pedido recebido, aguardando pagamento" |
| `PaymentFailed` | "Pagamento recusado" |
| `TicketsIssued` | "Seus ingressos chegaram" |
| `ReservationExpired` | "Sua reserva expirou" |
| `EventCanceled` | "Evento cancelado, reembolso em andamento" |

**Obrigações / regras**
- Envio simulado (log ou MailHog/Mailpit para ver os e-mails).
- Falha no envio **não** pode afetar a compra — retry e, depois, **Dead Letter Queue**.
- Precisa de dados do usuário (nome/e-mail): ou chama o Identity, ou mantém uma cópia local a partir de `UserRegistered` (decidir e justificar).

---

### 3.9 Search Service

**Responsabilidade:** busca de eventos publicados (texto, cidade, categoria, data). É um **read model (CQRS)**.

**Banco:** Elasticsearch (ou OpenSearch)

**Endpoints (síncrono)**
| Método | Rota | Descrição |
|---|---|---|
| GET | `/v1/search/events?q=&city=&category=&from=&to=` | Busca paginada |

**Eventos consumidos:** `EventPublished`, `EventUpdated`, `EventCanceled`

**Obrigações / regras**
- Nunca escreve em outro serviço; só reflete eventos.
- Aceitar **consistência eventual** (um evento recém-publicado pode demorar alguns segundos para aparecer).
- Ter um comando de **reindexação** completa (reprocessar eventos ou buscar tudo do Catalog).

---

## 4. Catálogo de eventos (mensageria)

**Envelope padrão de todas as mensagens:**
```json
{
  "messageId": "uuid",
  "type": "PaymentApproved",
  "version": 1,
  "occurredAt": "2026-09-26T12:00:00Z",
  "correlationId": "uuid",
  "source": "payments-service",
  "payload": { }
}
```

| Evento | Publicado por | Consumido por |
|---|---|---|
| `UserRegistered` | Identity | Notifications |
| `EventCreated` / `EventUpdated` | Catalog | Search |
| `EventPublished` | Catalog | Inventory, Search |
| `EventCanceled` | Catalog | Inventory, Orders, Payments, Tickets, Search, Notifications |
| `SeatsReserved` | Inventory | (auditoria/opcional) |
| `ReservationExpired` | Inventory | Orders, Notifications |
| `SeatsReleased` / `SeatsConfirmed` | Inventory | (auditoria/opcional) |
| `OrderCreated` | Orders | Payments, Notifications |
| `OrderConfirmed` | Orders | Inventory, Tickets |
| `OrderCanceled` | Orders | Inventory, Notifications |
| `PaymentApproved` | Payments | Orders |
| `PaymentFailed` | Payments | Orders, Notifications |
| `PaymentRefunded` | Payments | Tickets, Notifications |
| `TicketsIssued` | Tickets | Notifications |
| `TicketUsed` | Tickets | (métricas/opcional) |

---

## 5. Fluxo principal: compra de ingresso (Saga)

### Caminho feliz
```
Cliente ──POST /orders──▶ Gateway ──▶ Orders
                                        │ 1. gRPC Catalog: preço + evento publicado?     (síncrono)
                                        │ 2. gRPC Inventory: ReserveSeats (10 min)        (síncrono)
                                        │ 3. salva pedido AWAITING_PAYMENT + outbox
                                        │ 4. responde 201 ao cliente
                                        ▼
                               [OrderCreated] ──▶ Payments ──▶ gateway fake ──▶ webhook
                                                                   │
                                                          [PaymentApproved]
                                                                   ▼
                                                  Orders → CONFIRMED → [OrderConfirmed]
                                                                   │
                                              ┌────────────────────┴───────────────────┐
                                              ▼                                        ▼
                                  Inventory: assentos SOLD                 Tickets: emite ingressos
                                                                                       │
                                                                              [TicketsIssued]
                                                                                       ▼
                                                                    Notifications: e-mail ao cliente
```

### Caminhos de falha (compensação)
| Situação | O que acontece |
|---|---|
| Assento indisponível no passo 2 | Pedido não é criado, retorna `409 Conflict` |
| Catalog fora do ar | Circuit breaker abre, retorna `503`, nada é reservado |
| Pagamento recusado | `PaymentFailed` → Orders `CANCELED` → `OrderCanceled` → Inventory libera assentos |
| Pagamento não chega em 10 min | Inventory publica `ReservationExpired` → Orders `EXPIRED` |
| Pagamento aprovado **depois** de expirar | Orders rejeita a transição → dispara reembolso (`PaymentRefunded`) |
| Evento cancelado pelo organizador | Todos os serviços reagem a `EventCanceled`: cancela pedidos, reembolsa, invalida ingressos |

---

## 6. Padrões obrigatórios

| Padrão | Onde aplicar | Por quê |
|---|---|---|
| **Database per Service** | Todos | Isolamento e autonomia |
| **Saga orquestrada** | Orders | Transação distribuída sem 2PC |
| **Transactional Outbox** | Orders, Payments, Catalog | Não perder eventos se o broker cair |
| **Idempotent Consumer** | Todos os consumidores | Mensagens podem chegar duplicadas |
| **Idempotency-Key** | `POST /orders`, webhook de pagamento | Cliente/gateway podem reenviar |
| **Lock otimista / distribuído** | Inventory | Evitar venda dupla |
| **Retry + backoff exponencial** | Consumidores, Payments | Falhas temporárias |
| **Dead Letter Queue** | Todos os consumidores | Isolar mensagens que sempre falham |
| **Circuit Breaker + Timeout** | Chamadas síncronas do Orders | Não derrubar tudo em cascata |
| **CQRS** | Search | Leitura otimizada separada da escrita |
| **Distributed Tracing** | Todos | Enxergar uma requisição atravessando serviços |

---

## 7. Infraestrutura

Tudo via `docker-compose` na raiz:

| Componente | Uso |
|---|---|
| PostgreSQL (uma instância por serviço, ou um container com bancos separados e usuários distintos) | Identity, Inventory, Orders, Payments |
| MongoDB | Catalog, Tickets (bancos separados) |
| Redis | Locks e TTL do Inventory |
| RabbitMQ **ou** Kafka | Mensageria |
| Elasticsearch | Search |
| Jaeger | Tracing |
| Prometheus + Grafana | Métricas (opcional, fase final) |
| Mailpit | Visualizar e-mails do Notifications |

**Estrutura sugerida do repositório**
```
tickethub/
├── docker-compose.yml
├── PROJETO.md
├── gateway/
├── services/
│   ├── identity/
│   ├── catalog/
│   ├── inventory/
│   ├── orders/
│   ├── payments/
│   ├── tickets/
│   ├── notifications/
│   └── search/
├── contracts/          # schemas dos eventos (JSON Schema) e .proto do gRPC
└── tests/
    ├── e2e/
    └── load/           # scripts k6 para teste de concorrência
```

---

## 8. Roteiro de desenvolvimento

| Fase | Entregas | Foco de aprendizado |
|---|---|---|
| **1** | docker-compose, Gateway, Identity, Catalog | Serviços independentes, JWT, banco isolado |
| **2** | Inventory, Orders (reserva síncrona) | gRPC/REST entre serviços, concorrência, timeout |
| **3** | Broker, Payments, Saga completa com compensação | Comunicação assíncrona, consistência eventual |
| **4** | Tickets, Notifications | Coreografia de eventos, múltiplos consumidores |
| **5** | Outbox, idempotência, DLQ, circuit breaker | Resiliência |
| **6** | Search (CQRS), Jaeger, métricas | Read models, observabilidade |

---

## 9. Cenários de teste para validar o aprendizado

- [ ] **Concorrência:** 200 requisições simultâneas para o último assento → exatamente **1** pedido criado.
- [ ] **Idempotência HTTP:** enviar `POST /orders` duas vezes com a mesma `Idempotency-Key` → um único pedido.
- [ ] **Webhook duplicado:** gateway fake manda o mesmo webhook 2x → um único `PaymentApproved`, ingressos não duplicados.
- [ ] **Broker fora do ar:** derrubar o RabbitMQ/Kafka, criar pedidos, subir de novo → eventos são entregues (outbox).
- [ ] **Serviço consumidor fora do ar:** derrubar o Tickets, aprovar pagamentos, subir de novo → ingressos são emitidos.
- [ ] **Expiração:** não pagar em 10 min → assento volta a ficar disponível e pedido fica `EXPIRED`.
- [ ] **Pagamento tardio:** aprovar pagamento após expirar → reembolso automático.
- [ ] **Circuit breaker:** derrubar o Catalog → Orders responde rápido com 503, sem travar.
- [ ] **Mensagem envenenada:** publicar payload inválido → vai para DLQ após N tentativas, fila continua andando.
- [ ] **Cancelamento de evento:** cancelar um evento com vendas → pedidos cancelados, reembolsos feitos, ingressos invalidados, e-mails enviados.
- [ ] **Tracing:** abrir o Jaeger e ver uma compra completa atravessando Gateway → Orders → Catalog → Inventory → Payments → Tickets → Notifications com o mesmo trace.
