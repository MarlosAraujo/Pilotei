# 0004 — Ambiente de desenvolvimento, containers e cache do app

- **Data:** 27/09/2026
- **Status:** aprovada (pontos em aberto listados abaixo)
- **Branch:** `003-setup-infra`

## Contexto
A decisão `0003-microservices-kafka.md` deixou como PROPOSTA o modo híbrido dos serviços, o endereço do emulador, o Docker Compose e o Kafka em KRaft. A arquitetura técnica do `CLAUDE.md` também tinha como PROPOSTA o Room como cache e a Google Play Billing.

## Decisão
1. **Serviços em modo híbrido (HTTP + Kafka).** Cada serviço NestJS conecta ao Kafka e abre sua porta HTTP (7000–7600) para health check e, se útil, documentação interna. Só o `gateway` atende os apps.
2. **Desenvolvimento no WSL com Docker Compose.** Kafka, Postgres e todos os microservices rodam em containers, orquestrados pelo Docker Compose, no Docker Engine instalado direto no WSL2 (Ubuntu 24).
3. **Kafka em modo KRaft**, sem ZooKeeper.
4. **Hospedagem por containers, sem amarrar a um provedor.** As mesmas imagens Docker do desenvolvimento são publicadas no provedor escolhido (exemplos: Azure, AWS, Oracle Cloud, Hetzner). A configuração muda por variáveis de ambiente, não por código.
5. **App: Room só como cache de leitura** (o backend é a fonte da verdade, ver `0001-backend-fonte-da-verdade.md`). **Assinatura do Pilotei Driver pela Google Play Billing.**

## Consequências
- Cada serviço precisa de um `Dockerfile` próprio e de configuração só por variáveis de ambiente (endereço do Kafka, URL do Postgres, porta), para rodar igual no WSL e na nuvem.
- O health check HTTP serve tanto ao Docker Compose quanto ao orquestrador do provedor.
- O Kafka gerenciado (por exemplo, Amazon MSK, Azure Event Hubs com API Kafka, Confluent Cloud) ou um Kafka próprio em container é escolha da hospedagem. PRECISA VALIDAR custo e compatibilidade de cada opção.
- O Room não guarda lançamentos pendentes: sem internet, o app só consulta.

## Em aberto
1. **Emulador → gateway (`http://10.0.2.2:7000`):** continua PRECISA VALIDAR. O Android Studio e o emulador ainda serão instalados no Windows para o teste.
2. **Provedor de hospedagem:** a definir (Azure, AWS, Oracle Cloud, Hetzner ou outro).
3. ~~Catálogo de tópicos Kafka e autenticação.~~ Definidos em `0006-catalogo-topicos-kafka.md` e `0005-autenticacao.md`.

## Referências
- Microservices e Kafka: `docs/decisoes/0003-microservices-kafka.md`
- Stack: `docs/decisoes/0002-stack-apps-e-backend.md`
