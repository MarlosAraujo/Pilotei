# 0007 — Monorepo do backend e domínio de Métricas

- **Data:** 03/10/2026
- **Status:** Aprovado (03/10/2026)
- **Branch:** `004-setup-monorepo-backend`

## Contexto
A `0006-catalogo-topicos-kafka.md` deixou em aberto o local e a ferramenta do monorepo do backend. O próximo passo do roadmap é montar a estrutura e começar pelo domínio de Métricas (custo/km, resultados e ganho real por hora), com testes unitários. Antes do código, esta decisão fixa a estrutura e as fórmulas.

## Decisão

### 1. Repositório
- **Repositório próprio e privado:** `MarlosAraujo/Pilotei-Backend`. Contém todos os microservices e bibliotecas do backend que atende o app Pilotei Driver (e, no pós-MVP, o Pilotei APP).
- Este repositório (`Pilotei`) continua com o contexto, as especificações, as decisões e o protótipo. As decisões do backend continuam sendo registradas aqui, em `docs/decisoes/`.
- Mesmo fluxo de trabalho: `main` protegido, `develop` como base, branches numeradas e PR para `develop`. A numeração do `Pilotei-Backend` é independente (`001-...`, `002-...`).

### 2. Ferramenta: Nest CLI em modo monorepo
- CONFIRMADO (documentação do NestJS): o modo monorepo tem um único `package.json` e um único `node_modules` na raiz, `nest-cli.json` com `"monorepo": true` e a lista de `projects`. Os serviços ficam em `apps/` (`nest generate app`) e as bibliotecas em `libs/` (`nest generate library`), com prefixo de import configurável.
- **Prefixo das bibliotecas:** `@pilotei/` (ex.: `@pilotei/contracts`, como previsto na `0006`).
- Cada serviço gera seu próprio build e sua própria imagem Docker (`nest build <serviço>`).
- **Versões (PROPOSTA):** Node 24 LTS, NestJS na última versão estável, TypeScript estrito, Prisma v7 (decisão `0003`), Jest para testes (padrão do NestJS). PRECISA VALIDAR as versões exatas na criação do projeto.
- **Gerenciador de pacotes:** pnpm.

### 3. Estrutura
```
Pilotei-Backend/
├── apps/
│   ├── gateway/        7000  BFF HTTP, autenticação (valida o JWT)
│   ├── core/           7100  Identidade + Assinaturas
│   ├── vehicle/        7200  Veículos, Custos, Ganhos, Importação
│   ├── reports/        7300  Métricas + Relatórios
│   └── notification/   7400  Config remota + Notificações
│       (copilot 7500 no MVP 2; trip 7600 no pós-MVP)
├── libs/
│   ├── contracts/      @pilotei/contracts: envelope, eventos e RPC (tipos)
│   ├── kafka/          @pilotei/kafka: outbox, idempotência por eventId, DLQ
│   ├── common/         @pilotei/common: config por env, health check, dinheiro (centavos), datas (UTC ↔ America/Sao_Paulo)
│   └── metrics/        @pilotei/metrics: fórmulas puras do domínio de Métricas
├── docker/
│   ├── compose.yaml    Postgres, Kafka (KRaft) e serviços
│   └── kafka/create-topics.sh
├── nest-cli.json
├── package.json
└── tsconfig.json
```
- **Prisma por serviço:** cada serviço tem `apps/<serviço>/prisma/` com o próprio schema e o próprio client gerado, apontando para o seu schema do Postgres (`core`, `vehicle`, `reports`, `notification`). Nenhum serviço importa o client de outro (regra da `0003`). PRECISA VALIDAR na montagem: a forma de configurar vários schemas Prisma no mesmo repositório no Prisma v7 (`prisma.config.ts` por serviço).
- **Por que `@pilotei/metrics` é uma biblioteca:** as fórmulas não dependem de banco, Kafka nem NestJS. Ficam testáveis isoladamente, e o `copilot` (MVP 2) pode reutilizá-las. Só o `reports` grava e publica os resultados.

### 4. Fórmulas do domínio de Métricas
Todas as entradas e saídas de dinheiro são em centavos (inteiro). Distâncias em metros, tempos em minutos.

| Métrica | Fórmula |
|---|---|
| **Custo/km** | combustível/km + manutenção e pneus/km + custos fixos/km |
| **Faturamento** | soma de `TripImport.net_amount` + `Income` + `PlatformDaySummary.net_amount`, sem contar em dobro (o PDF da Uber substitui os dias manuais da Uber no mesmo período) |
| **Resultado econômico** | faturamento − KM × custo/km vigente no dia |
| **Resultado de caixa** | faturamento − valores efetivamente pagos no período (abastecimentos, manutenções, despesas, custos fixos pagos) |
| **Faturamento por km** | faturamento ÷ KM |
| **Ganho real por hora** | resultado econômico ÷ horas online |

Conferência com os números das telas: 402,60 − 200 × 0,98 = 206,60; 206,60 ÷ 7,5 h = 27,546 → **R$ 27,55/h** ✓.

#### 4.1 Componentes do custo/km
| Componente | Estimado (informado pelo motorista) | Observado (dados reais) |
|---|---|---|
| **Combustível/km** | preço do litro ÷ consumo (km/l) informado | soma dos abastecimentos entre dois **tanques cheios** ÷ KM rodados entre eles |
| **Manutenção e pneus/km** | valor por km informado | soma das manutenções na janela ÷ KM rodados na janela |
| **Custos fixos/km** | custos fixos mensais ÷ meta de KM por mês (`Goal`) | custos fixos mensais ÷ média de KM por mês observada |

- **Rateio dos custos fixos pelo KM total** (trabalho + uso pessoal), porque o veículo gera esses custos rodando para qualquer fim.
- **Janela do observado:** últimos 90 dias, para manutenção e KM médio por mês.

- **KM do dia:** odômetro final − odômetro inicial (`OdometerReading`). Inclui o KM vazio. Se faltar odômetro, usar a soma dos KM de `PlatformDaySummary` e marcar a confiança como baixa.
- Cada componente escolhe a sua base de forma independente: o combustível pode já ser observado enquanto a manutenção ainda é estimada.

#### 4.2 Base e nível de confiança
| Base | Quando | Confiança |
|---|---|---|
| **Estimado** | só valores informados | baixa |
| **Observado** | ≥ 2 tanques cheios e ≥ 500 km reais | média |
| **Observado** | ≥ 30 dias e ≥ 1.500 km reais, com manutenção registrada | alta |
| **Personalizado** | o motorista ajustou à mão algum componente | a do observado original, com a marca "ajustado por você" |

- A confiança do custo/km é a **menor** entre as dos seus componentes.
- **Personalizado = ajuste manual.** O motorista pode ajustar qualquer componente (ex.: troca de pneus prevista). O app mostra que o valor foi ajustado, e o observado original continua guardado e visível, para comparação. O ajuste pode ser desfeito a qualquer momento. Sazonalidade e padrão de uso ficam para o MVP 3 (Inteligência).
- O `CostSnapshot` guarda os componentes, a base e a confiança de cada um, a janela usada e a data de vigência. O resultado de um dia usa o snapshot vigente naquele dia.

#### 4.3 Precisão e arredondamento
- Custo/km guardado em **centésimos de centavo por km** (inteiro; ex.: R$ 0,98/km = `9800`), para não perder precisão na multiplicação pelo KM.
- Contas intermediárias em inteiros; arredondamento **meio para cima** só no resultado final em centavos.
- Divisão por zero (0 km ou 0 h): a métrica fica "sem dado", nunca zero ou infinito.

#### 4.4 Horas online nos dias importados pelo PDF
- O PDF da Uber não traz o tempo online. O motorista informa as horas online de cada dia no próprio dia ou depois, consultando o tempo online que o app Uber Driver mostra. Pode informar na prévia da importação ou no Financeiro.
- O Pilotei não lê esse dado do app Uber Driver: o motorista digita as horas.
- Enquanto não informar, o ganho real por hora desses dias fica "sem dado". O app nunca estima o tempo pelos horários das corridas.
- Nos períodos com dias com e sem horas informadas, o ganho real por hora considera só os dias com horas, e a tela mostra quantos dias ficaram de fora.

### 5. Escopo das próximas branches no `Pilotei-Backend`
1. `001-setup/estrutura`: monorepo Nest, `libs/common`, `libs/contracts`, os 5 serviços do MVP 1 só com health check, Docker Compose (Postgres + Kafka KRaft + script de tópicos), lint e CI de testes.
2. `002-metricas`: `libs/metrics` com as fórmulas da seção 4 e testes unitários (TDD), incluindo o exemplo das telas.
3. Depois: `reports` usando `@pilotei/metrics`, com `CostSnapshot`, `Goal` e os consumidores dos eventos do `vehicle`.

## Consequências
- Um único `package.json`: todos os serviços usam as mesmas versões de dependências. Isso simplifica as atualizações, mas uma atualização afeta todos os serviços ao mesmo tempo.
- O CI precisa construir e testar só os serviços afetados por um commit, ou todos, enquanto o projeto for pequeno.
- As decisões ficam neste repositório e o código no `Pilotei-Backend`. Cada PR do backend deve citar a decisão correspondente.

## Em aberto
Nenhum ponto. Os três pontos levantados no rascunho foram definidos em 03/10/2026: personalizado = ajuste manual (4.2), limites de confiança aprovados (4.2) e horas online informadas pelo motorista (4.4).

## Referências
- Microservices e Kafka: `docs/decisoes/0003-microservices-kafka.md`
- Ambiente: `docs/decisoes/0004-ambiente-e-containers.md`
- Catálogo de tópicos: `docs/decisoes/0006-catalogo-topicos-kafka.md`
- Domínios e entidades: `docs/backend/dominios.md`, `docs/backend/glossario-entidades.md`
