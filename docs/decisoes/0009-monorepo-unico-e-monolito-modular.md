# 0009 — Monorepo único, monólito modular e revisão do Prompt Mestre

- **Data:** 10/10/2026
- **Status:** Aprovado (10/10/2026)
- **Branch:** `008-readme-prompt-mestre`

## Contexto
O responsável pelo produto escreveu um Prompt Mestre de desenvolvimento para o Claude Code (`docs/prompt-mestre.md`). O rascunho divergia de várias decisões anteriores (0001 a 0008). Os conflitos foram revistos ponto a ponto, em 16 itens, e o prompt foi reescrito com as escolhas abaixo. O foco é reduzir o custo de infraestrutura e de manutenção no MVP: um repositório, um processo de backend, uma VPS pequena.

## Decisão

### 1. Monorepo único
- **App e backend no mesmo repositório:** `Pilotei/Pilotei-Driver` (privado), com `apps/pilotei-driver` (React Native + Expo), `apps/pilotei-backend` (NestJS) e `packages/` (PROPOSTA: leitor do PDF da Uber e tipos compartilhados).
- O `Pilotei-Backend` será **arquivado** depois que o que serve for migrado (principalmente o `libs/metrics`, com os testes).
- Este repositório (`Pilotei`) guarda o histórico de planejamento (decisões 0001 a 0009, protótipo e especificações). As próximas decisões passam a ser **ADRs no monorepo**, em `docs/architecture-decisions/ADR-00N-*.md`, com o `AGENTS.md` como guia do Claude Code.

### 2. Backend em monólito modular
- **NestJS em monólito modular**, sem microservices, sem Kafka e sem Kubernetes.
- Módulos: `auth`, `users`, `drivers`, `vehicles`, `rides`, `earnings`, `expenses`, `goals`, `imports`, `metrics`, `subscriptions`, `licenses` e `notifications`.
- Camadas: Controller → Use Case → regras de domínio → Repository → Prisma → PostgreSQL. As fórmulas de métricas continuam puras, sem banco nem NestJS.
- PostgreSQL e Prisma 7. API REST em `/api/v1`, com Swagger.

### 3. App
- **React Native + Expo** (development build), como na 0008. Copiloto em Kotlin por Expo Modules.
- **Navegação com 9 seções:** Dashboard, Corridas, Ganhos, Despesas, Metas, Veículos, Análises, Configurações e Assinatura. O protótipo em `design/` (menu Início, Financeiro e Relatórios) precisa ser revisto.

### 4. Assinatura
- **Mensal: R$ 10,99. Anual: R$ 97,99** (equivale a R$ 8,17/mês, cerca de 26% mais barato). **7 dias de teste grátis**, só para quem nunca assinou. Google Play Billing, validação no backend.

### 5. Dados
- **Dinheiro em `Decimal`** (PostgreSQL/Prisma) e representação decimal segura no TypeScript, no lugar dos centavos inteiros.
- **Entidades do Prompt Mestre:** `User`, `Driver` (perfil separado do usuário), `Vehicle`, `Ride` (no lugar de `TripImport`), `Earning`, `Expense`, `Goal`, `Subscription`, `License` (com entitlements), `Notification` e `RefreshToken`. PROPOSTA: `OtpCode` (login sem senha) e `ImportBatch` (importação do PDF); o dia lançado à mão como um tipo de `Earning`.
- **Plataformas:** `UBER`, `NINE_NINE`, `INDRIVE`, `PRIVATE` e `OTHER`.

### 6. Hospedagem
- **VPS Hetzner CPX22** (2 vCPU, 4 GB RAM, 80 GB SSD), com proxy reverso e HTTPS, backend e PostgreSQL em Docker. O PostgreSQL fica privado. Fecha a pendência 8.

### 7. Mantidos
- **Internet obrigatória para lançar** e backend como fonte da verdade (0001).
- **Login com Google ou e-mail + OTP, sem senha**, pelo Resend (0005), com as mesmas regras de OTP e tokens. O JWT passa a ser validado pelo próprio backend.
- **Importação do PDF semanal da Uber no MVP 1**, com o parser em TypeScript + pdf.js (0008).
- **Copiloto no MVP 2**, começando só pela Uber, com o app de teste de acessibilidade antes.
- Todas as regras de produto: custo/km com confiança, resultado econômico e de caixa, ganho real por hora, marcas só em texto, aviso de não afiliação.

## O que muda nas decisões anteriores
| Decisão | Antes | Agora |
|---|---|---|
| `0002` | Backend em microservices | Monólito modular NestJS |
| `0003` | 7 serviços, Kafka, um schema por serviço | **Substituída.** Um backend, um banco; os serviços viram módulos |
| `0004` | Modo híbrido HTTP + Kafka; Kafka KRaft no Compose; hospedagem sem provedor | Compose só com `pilotei-backend` e `postgres`; Hetzner CPX22 |
| `0005` | Autenticação no `core`; JWT validado no `gateway`; eventos `user.deleted` | Módulo `auth` no monólito; sem eventos Kafka. Regras de login e tokens iguais |
| `0006` | Catálogo de tópicos Kafka | **Substituída.** Sem Kafka |
| `0007` | Repositório `Pilotei-Backend`; libs `contracts` e `kafka`; dinheiro em centavos | Monorepo `Pilotei-Driver`; sem `kafka`; dinheiro em `Decimal`. As fórmulas e regras de confiança continuam |
| `0008` | Repositório `Pilotei-Driver` só para o app | O mesmo repositório recebe também o backend |
| Modelo de dados | `TripImport`, `Income`, `PlatformDaySummary`, `NINETY_NINE` | `Ride`, `Earning`, `Driver`, `License`, `NINE_NINE` |
| Preço anual | R$ 99,90 (sugestão) | R$ 97,99 |
| Navegação | Início, Financeiro e Relatórios | 9 seções |
| Registro de decisões | `docs/decisoes/000N-*.md` neste repositório | ADRs em `docs/architecture-decisions/` no monorepo |

## Motivos
- **Custo baixo:** um processo de backend e um PostgreSQL cabem numa VPS de 4 GB. O Kafka sozinho consumiria boa parte da memória.
- **Menos peças no MVP:** sem outbox, DLQ, RPC por tópicos nem vários deploys.
- **Um repositório:** app e backend mudam juntos, compartilham tipos TypeScript e passam pelo mesmo CI.
- **Crescimento sem reescrita:** módulos com fronteiras claras podem virar serviços no futuro, se necessário.

## Consequências
- O código do `Pilotei-Backend` (estrutura de 5 serviços, `libs/kafka`, `libs/contracts`, Compose com Kafka) não é aproveitado; o `libs/metrics` é migrado e adaptado para `Decimal`.
- O PR #1 do `Pilotei-Driver` passa a ser a FASE 0/1 do monorepo, não só o projeto Expo com o leitor do PDF.
- O `CLAUDE.md` e o `README.md` deste repositório passam a apontar para o Prompt Mestre e para o monorepo.

## Em aberto
1. Precisão das colunas `Decimal` (o custo/km precisa de mais casas que o centavo), regra de arredondamento e biblioteca decimal no TypeScript.
2. Gerenciador de pacotes e de workspace do monorepo, e ferramentas de lint e teste.
3. Quais das 9 seções ficam no menu inferior.
4. Limite de aparelhos por conta (citado no rascunho, sem número).
5. Licença `FREE` do rascunho, que não combina com um produto só pago.
6. Onde roda a análise da oferta no Copiloto (proposta: no aparelho, com o custo/km do servidor).
7. Development build local ou EAS Build; pdf.js no Hermes (0008).

## Referências
- Prompt Mestre revisado: `docs/prompt-mestre.md`
- Decisões substituídas: `0003-microservices-kafka.md`, `0006-catalogo-topicos-kafka.md`
- Decisões alteradas: `0002`, `0004`, `0005`, `0007`, `0008`
