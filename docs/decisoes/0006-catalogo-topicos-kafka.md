# 0006 — Catálogo de tópicos Kafka e contratos

- **Data:** 27/09/2026
- **Status:** substituída pela `0009-monorepo-unico-e-monolito-modular.md` (10/10/2026)
- **Branch:** `003-setup-infra`

## Contexto
A decisão `0003-microservices-kafka.md` definiu Kafka entre os serviços, com eventos (pub/sub) e pedido-resposta do `gateway`. A `0005-autenticacao.md` definiu o envelope das mensagens e o `userId` como chave de partição. Faltava o catálogo: nomes, donos, consumidores e regras.

## Decisão

### 1. Regras gerais
| Ponto | Regra |
|---|---|
| **Nome dos tópicos de evento** | `<serviço>.<assunto>.v<N>`, minúsculas, palavras separadas por `-`. Ex.: `vehicle.cost.v1` |
| **Um tópico por assunto** | O tipo do evento vai no campo `type` do envelope (ex.: `fuel-entry.created`). Criar e apagar o mesmo registro chegam em ordem |
| **Envelope** | `{ eventId, type, version, occurredAt, userId, correlationId, payload }` (decisão 0005) |
| **Chave da mensagem** | `userId`, para manter a ordem por motorista |
| **Versão** | `version` sobe para campos novos (compatível). Mudança incompatível cria um tópico `v2`, que convive com o `v1` durante a migração |
| **Conteúdo** | JSON. Dinheiro em centavos; datas em UTC, ISO 8601. **Sem dados pessoais** (e-mail, nome): só IDs e valores |
| **Grupo de consumidores** | O nome do serviço (ex.: `reports`) |
| **Outbox** | O serviço grava o dado e o evento na mesma transação, numa tabela `outbox` do próprio schema; um publicador envia a tabela ao Kafka |
| **Idempotência** | Cada consumidor guarda os `eventId` processados e ignora repetidos (entrega pelo menos uma vez) |
| **Falhas** | Após 3 tentativas, a mensagem vai para `<tópico>.dlq` |
| **Criação dos tópicos** | Por script no Docker Compose; criação automática desligada. Desenvolvimento: 1 broker, 3 partições, sem réplica. Produção: PRECISA VALIDAR com o provedor |

### 2. Eventos do MVP 1
| Tópico | Eventos (`type`) | Consumidores |
|---|---|---|
| `core.user.v1` | `user.created`, `user.deleted` | todos (`user.deleted` apaga os dados do motorista em cada serviço) |
| `core.subscription.v1` | `subscription.updated` (estado, fim do teste, vencimento) | `notification` (aviso 2 dias antes do fim do teste) |
| `core.entitlement.v1` | `entitlement.changed` (pode lançar: sim/não). **Tópico compactado**: guarda o último estado por `userId` | `vehicle` (bloqueia lançamentos sem assinatura, sem consultar o `core`) |
| `vehicle.vehicle.v1` | `vehicle.created`, `vehicle.updated`, `vehicle.deleted` | `reports` |
| `vehicle.odometer.v1` | `odometer.recorded` | `reports` (KM), `notification` (manutenção próxima) |
| `vehicle.cost.v1` | `fuel-entry.*`, `maintenance.*`, `fixed-cost.*`, `expense.*` (`created`, `updated`, `deleted`) | `reports`; `notification` (manutenção prevista) |
| `vehicle.earnings.v1` | `trip-import.*`, `income.*`, `platform-day.*` | `reports` |
| `vehicle.import.v1` | `import-batch.completed`, `import-batch.reverted` | `reports` (recalcula o período) |
| `reports.cost-snapshot.v1` | `cost-snapshot.calculated` | nenhum no MVP 1; `copilot` no MVP 2 |

Depois do MVP 1: `copilot.offer.v1` (MVP 2) e os tópicos do `trip` (pós-MVP), definidos na modelagem.

### 3. Pedido e resposta (`gateway` ↔ serviços)
- Mantido o pedido-resposta do NestJS sobre Kafka (`send`), como na `0003`.
- **Nome:** `<serviço>.rpc.<ação>` (RPC = *Remote Procedure Call*: uma ordem que espera resposta). Ex.: `vehicle.rpc.fuel-entry.create`, `reports.rpc.dashboard.get`.
- A resposta vai para `<tópico>.reply`, criado pelo NestJS (CONFIRMADO).
- `rpc` separa pedidos (imperativo, um único serviço responde) de eventos (passado, vários consomem).
- A lista de ações sai na modelagem de cada serviço.

### 4. Contratos compartilhados
- **Backend em monorepo**, com o pacote **`@pilotei/contracts`**: tipos TypeScript do envelope, dos eventos e dos pedidos RPC, usados por todos os serviços.

## Consequências
- Cada schema de serviço ganha as tabelas `outbox` e de eventos processados.
- Um evento mal formado não trava o consumidor: vai para a DLQ, que precisa ser monitorada.
- Mudar um contrato em `@pilotei/contracts` afeta todos os serviços no mesmo commit, o que facilita achar quebras.
- Pedido-resposta sobre Kafka gera dois tópicos por ação e passa cada resposta pelo broker. Se a latência do dashboard incomodar, a alternativa registrada é HTTP interno entre `gateway` e serviços (mudaria a `0003`).

## Em aberto
1. Local do monorepo (pasta `backend/` neste repositório ou repositório próprio) e ferramenta (npm/pnpm workspaces, Nx, Turborepo). Definir na montagem da estrutura.
2. Partições e réplicas em produção, junto com o provedor de hospedagem.
3. Tópicos do MVP 2 e do pós-MVP.

## Referências
- Microservices e Kafka: `docs/decisoes/0003-microservices-kafka.md`
- Autenticação e envelope: `docs/decisoes/0005-autenticacao.md`
- Domínios: `docs/backend/dominios.md`
