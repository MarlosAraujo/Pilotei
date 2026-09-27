# 0003 — Microservices, Kafka e banco

- **Data:** 27/09/2026
- **Status:** aprovada (pontos em aberto listados abaixo)
- **Branch:** `002-setup-stack`

## Contexto
A decisão `0002-stack-apps-e-backend.md` definiu o backend em NestJS com microservices. Faltava agrupar os 11 domínios (`docs/backend/dominios.md`) em serviços e escolher a comunicação entre eles.

## Decisão

### Serviços
| Serviço | Porta | Domínios | Fase |
|---|---|---|---|
| `gateway` | 7000 | BFF: HTTP para os apps, com autenticação | MVP 1 |
| `core` | 7100 | Identidade + Assinaturas | MVP 1 |
| `vehicle` | 7200 | Veículos, Custos, Ganhos e Importação | MVP 1 |
| `reports` | 7300 | Métricas + Relatórios | MVP 1 |
| `notification` | 7400 | Config remota e flags + Notificações | MVP 1 |
| `copilot` | 7500 | Copiloto | MVP 2 |
| `trip` | 7600 | Corridas do Pilotei APP (pedido, despacho, preço, cobrança da taxa) | Pós-MVP |

### Comunicação
- **Apps → backend:** só pelo `gateway`, em HTTP. Os demais serviços não ficam expostos aos apps.
- **Entre serviços:** **Kafka** (`@nestjs/microservices`, transporte Kafka). **Todos os serviços se conectam ao mesmo cluster Kafka**, cada um com seus próprios tópicos.
- **Todo serviço publica e consome eventos** (pub/sub): cada um tem consumidores para os tópicos que lhe interessam.

### Banco
- **Um PostgreSQL único no início, com um schema por serviço** (`core`, `vehicle`, `reports`, `notification`, `copilot`, `trip`).
- **ORM: Prisma v7.** PROPOSTA: um schema Prisma por serviço, apontando para o schema do Postgres daquele serviço. Configuração exata da v7 PRECISA VALIDAR na documentação.
- Um serviço só lê e escreve no próprio schema. Dados de outro serviço chegam por evento ou por chamada via `gateway`/Kafka, nunca por consulta direta ao schema alheio.

### Ambiente de desenvolvimento
- **Backend no WSL2** (Ubuntu 24): **Kafka, Postgres e todos os microservices rodam em Docker**, com **Docker Engine instalado direto no WSL** (sem Docker Desktop).
- **Apps Kotlin no Windows**, com Android Studio e emulador.
- **Emulador → gateway:** o emulador chega ao `localhost` do Windows pelo endereço `10.0.2.2` (CONFIRMADO). O WSL2 encaminha as portas para o `localhost` do Windows por padrão (`localhostForwarding`, CONFIRMADO). O app usaria `http://10.0.2.2:7000`. PRECISA VALIDAR na máquina.

## Consequências
- **Porta com Kafka:** um serviço NestJS que só usa o transporte Kafka conecta ao broker e não abre porta HTTP. PROPOSTA: cada serviço roda como *aplicação híbrida* (HTTP + Kafka), usando a porta para health check e, se útil, documentação interna. Só o `gateway` atende os apps.
- **Pedido e resposta:** quando o `gateway` precisa de uma resposta (por exemplo, buscar o dashboard), usa o padrão pedido-resposta do Kafka no NestJS (`send` + tópico de resposta). É mais lento e mais complexo que HTTP direto; monitorar a latência.
- **Eventos exigem cuidado:** entrega pelo menos uma vez (consumidores idempotentes), chave de partição por `userId` para manter a ordem por motorista, e nomes de tópicos versionados.
- **Operação local mais pesada:** Kafka + Postgres + 5 serviços (7 no pós-MVP) em containers no WSL. As portas 7000–7600 são publicadas pelos containers.

## Em aberto
1. ~~PROPOSTA: Docker Compose, Kafka em modo KRaft (sem ZooKeeper). Modo híbrido (HTTP + Kafka) e `10.0.2.2:7000` também seguem como PROPOSTA.~~ Aprovados em `0004-ambiente-e-containers.md`, exceto `10.0.2.2:7000`, que segue PRECISA VALIDAR.
2. ~~Catálogo de tópicos e eventos (nomes, donos, formato das mensagens).~~ Definido em `0006-catalogo-topicos-kafka.md`.
3. Autenticação e hospedagem.

## Referências
- Stack: `docs/decisoes/0002-stack-apps-e-backend.md`
- Domínios: `docs/backend/dominios.md`
