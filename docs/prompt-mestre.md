# PILOTEI DRIVER — Prompt Mestre de Desenvolvimento para Claude Code

> Revisado em 10/10/2026, ponto a ponto, sobre o rascunho original. As escolhas desta revisão substituem as decisões 0003 (microservices e Kafka), 0006 (tópicos Kafka) e parte da 0002, 0004, 0007 e 0008. Elas devem ser registradas como ADR no monorepo (FASE 0).
>
> Legenda: **CONFIRMADO** (fato verificado), **PROPOSTA** (sugestão a aprovar), **PRECISA VALIDAR** (depende de teste ou de consulta).

Você é o principal engenheiro de software responsável por projetar, implementar, testar, documentar e preparar para produção o aplicativo **Pilotei Driver**.

O objetivo é construir uma plataforma profissional para motoristas de aplicativo, inicialmente só no Android, que permita acompanhar corridas, ganhos, despesas, resultado real, metas e produtividade. No MVP 2, ela usa recursos de acessibilidade do Android para auxiliar o motorista durante o uso dos apps de transporte (Copiloto).

O projeto é desenvolvido em um **único monorepositório**, `https://github.com/Pilotei/Pilotei-Driver.git`, com o app e o backend.

---

# 1. OBJETIVO DO PRODUTO

Criar o **Pilotei Driver**, que permita:

- acompanhar ganhos por plataforma;
- registrar corridas e dias de trabalho;
- importar o relatório semanal da Uber em PDF;
- calcular o resultado econômico e o resultado de caixa;
- calcular o custo por quilômetro, com nível de confiança;
- calcular o ganho real por hora;
- controlar combustível, manutenção (inclusive preventiva) e despesas;
- acompanhar metas;
- acompanhar o desempenho diário, semanal e mensal;
- cadastrar veículos;
- analisar a produtividade;
- controlar e validar a assinatura;
- manter o histórico de uso;
- no MVP 2: ler a oferta na tela do app de transporte e mostrar a análise em overlay (Copiloto).

Plataformas iniciais: **Uber, 99, inDrive e corridas particulares**.

A arquitetura deve permitir novas plataformas sem reescrever o núcleo.

Não confundir nunca: receita ≠ lucro; R$/km ≠ lucro/km; oferta ≠ corrida; estimativa ≠ real; resultado de caixa ≠ resultado econômico.

Marcas de terceiros (Uber, 99, inDrive) aparecem **só como palavra**, sem logo, ícone ou cores. Aviso obrigatório: "O Pilotei não é afiliado à Uber, 99 ou inDrive."

---

# 2. MODELO COMERCIAL

- Nome: **Pilotei Driver**.
- Assinatura pela **Google Play Billing**: um produto de assinatura com dois planos.
  - **Mensal: R$ 10,99.**
  - **Anual: R$ 97,99.**
  - **7 dias de teste grátis**, só para quem nunca assinou.
- Os preços exibidos vêm da Google Play (detalhes do produto), nunca fixos no código.
- Não implementar pagamento com cartão dentro do app. Não criar gateway próprio para a assinatura.
- A assinatura é validada pelo backend. O app nunca trata uma informação local como prova de assinatura válida.
- Assinatura expirada: consulta e exportação liberadas; novos lançamentos exigem assinatura.
- Exclusão de conta no Perfil, como a Google Play exige.

---

# 3. OBJETIVO FINANCEIRO DO PROJETO

Premissas iniciais:

- VPS **Hetzner CPX22** (2 vCPU, 4 GB RAM, 80 GB SSD), cerca de US$ 19,99/mês;
- Claude Code: R$ 110/mês;
- domínio: R$ 40/ano;
- PostgreSQL na própria VPS;
- NestJS no backend;
- Google Play Console: US$ 25 de taxa única;
- preço: R$ 10,99/mês ou R$ 97,99/ano.

O sistema deve operar com baixo custo de infraestrutura:

- não criar microserviços;
- não usar Kafka nem mensageria distribuída;
- não usar Kubernetes;
- não criar arquitetura distribuída desnecessária.

Usar um **monólito modular NestJS**, que permita crescer sem reescrever o sistema.

---

# 4. ESTRUTURA DO MONOREPOSITÓRIO

```text
Pilotei-Driver/
│
├── apps/
│   ├── pilotei-driver/          # React Native + Expo (development build)
│   │   ├── src/
│   │   ├── modules/             # Expo Modules em Kotlin (Copiloto, MVP 2)
│   │   └── app.config.ts
│   │
│   └── pilotei-backend/         # NestJS, monólito modular
│       ├── src/
│       │   ├── modules/
│       │   │   ├── auth/
│       │   │   ├── users/
│       │   │   ├── drivers/
│       │   │   ├── vehicles/
│       │   │   ├── rides/
│       │   │   ├── earnings/
│       │   │   ├── expenses/
│       │   │   ├── goals/
│       │   │   ├── imports/
│       │   │   ├── metrics/
│       │   │   ├── subscriptions/
│       │   │   ├── licenses/
│       │   │   └── notifications/
│       │   ├── common/
│       │   ├── database/
│       │   └── main.ts
│       ├── prisma/
│       ├── test/
│       ├── Dockerfile
│       └── package.json
│
├── packages/                    # PROPOSTA: código TypeScript compartilhado
│   ├── uber-pdf-parser/         # leitor do PDF semanal da Uber (TS puro)
│   └── contracts/               # tipos de DTO compartilhados app ↔ API
│
├── docs/
├── docker/
├── scripts/
├── .github/workflows/
├── docker-compose.yml
├── README.md
├── AGENTS.md
└── .gitignore
```

Não alterar essa separação sem justificativa técnica registrada em ADR.

PRECISA DEFINIR (FASE 0, em ADR): gerenciador de pacotes e de workspace (npm, pnpm ou outro) e ferramentas de lint e teste.

---

# 5. TECNOLOGIAS

## App (Android no MVP; iOS no pós-MVP, sem o Copiloto)

- React Native + Expo, em **development build** (sem Expo Go);
- TypeScript estrito;
- navegação: Expo Router ou React Navigation (PROPOSTA, decidir na FASE 5);
- cache só de leitura: TanStack Query persistido ou `expo-sqlite` (PROPOSTA);
- `expo-document-picker` (PDF pelo seletor do Android, sem permissão de armazenamento);
- `expo-web-browser` (portal da Uber em Custom Tab, nunca em WebView);
- `@react-native-google-signin/google-signin` (login com Google);
- `expo-iap` ou `react-native-iap` (Google Play Billing) (PROPOSTA);
- **Kotlin, por Expo Modules e config plugins**, só para o código nativo: AccessibilityService e overlay (MVP 2) e, se o pdf.js falhar no Hermes, a extração do PDF com PdfBox-Android.

PRECISA VALIDAR: development build local (Android SDK no WSL) ou EAS Build.

Evitar dependências desnecessárias. Priorizar APIs oficiais do Android e do Expo.

---

# 6. BACKEND

- Node.js, NestJS, TypeScript estrito;
- Prisma 7 e PostgreSQL;
- JWT;
- class-validator;
- Swagger/OpenAPI;
- Docker.

O backend é um **monólito modular**. Cada módulo tem suas próprias responsabilidades:

```text
modules/
└── rides/
    ├── rides.module.ts
    ├── controllers/
    ├── providers/
    ├── dto/
    ├── entities/
    ├── use-cases/
    └── repositories/
```

Manter as regras de negócio independentes do framework sempre que possível. As fórmulas de métricas ficam em código puro, sem banco nem NestJS (aproveitar o `libs/metrics` do `Pilotei-Backend`, com seus testes).

---

# 7. ARQUITETURA DO BACKEND

```text
Controller
    ↓
Use Case
    ↓
Domain/Business Rules
    ↓
Repository
    ↓
Prisma
    ↓
PostgreSQL
```

- Controllers e repositories não contêm regras de negócio.
- Use cases orquestram as operações.
- As regras financeiras são testáveis sem depender do NestJS.
- **O backend é a fonte da verdade.** As métricas (custo/km, resultados, ganho real por hora) são calculadas no servidor.

---

# 8. MÓDULO AUTH

Autenticação própria, **sem senha**:

- login com **Google** (token de ID obtido no app, validado com `google-auth-library`);
- login com **e-mail + código OTP**, enviado pelo **Resend** (domínio `pilotei.app.br`);
- Google e OTP com o mesmo e-mail caem no mesmo `User`;
- logout;
- refresh token;
- controle e revogação de sessão;
- proteção contra força bruta.

Regras:

```text
OTP           -> 6 dígitos, 10 min, 5 tentativas, reenvio após 60 s, guardado como hash
access token  -> JWT de 15 min (sub = userId)
refresh token -> 30 dias, trocado a cada uso, guardado como hash
```

Sem senha, não há cadastro de senha, troca de senha nem recuperação de senha.

---

# 9. MÓDULO USERS

- usuário: nome, e-mail, telefone opcional, status;
- datas de criação e atualização;
- preferências, timezone (padrão America/Sao_Paulo) e configurações do app;
- exclusão de conta e dos dados (LGPD e Google Play).

---

# 10. MÓDULO DRIVERS

O conceito de usuário e o de motorista são separados. Um usuário pode ter um perfil de motorista.

- criação e atualização do motorista;
- configurações e preferências;
- plataformas usadas;
- metas;
- configurações financeiras: valor mínimo por km e por hora.

---

# 11. MÓDULO VEHICLES

- marca, modelo, ano, placa;
- combustível, consumo médio, preço do combustível;
- custo estimado de manutenção;
- custo por km (estimado, observado e personalizado, ver seção 13);
- seguro, IPVA, financiamento, outros custos fixos;
- uso pessoal (sim/não).

Permitir múltiplos veículos por usuário, com um veículo principal.

---

# 12. MÓDULO RIDES

Representa uma corrida de outra plataforma, importada ou lançada à mão.

```text
id
driverId
vehicleId
platform
category
externalRideId        # ou hash contra duplicatas (PDF da Uber não traz ID)
startedAt
finishedAt
origin
destination
distanceKm
durationMinutes
grossAmount
platformFee
netAmount
tips
status
source                # FILE ou MANUAL
importBatchId
metadata
createdAt
updatedAt
```

Plataformas:

```text
UBER
NINE_NINE
INDRIVE
PRIVATE   # corridas particulares
OTHER
```

99Pop é categoria da 99, não plataforma.

Não espalhar regras da Uber pelo código. Criar uma abstração de plataforma.

**Dia lançado à mão** ("Adicionar dia": data, plataforma, ganho, km, horas online, nº de corridas) não cria corridas inventadas. PROPOSTA: guardar como `Earning` do tipo resumo diário (seção 13).

---

# 13. MÓDULO EARNINGS

Responsável pelos ganhos e pelos resultados.

Tipos de `Earning` (PROPOSTA): corrida (vinculada à `Ride`), bônus/promoção e resumo diário manual.

Calcular:

- faturamento (ganho bruto e líquido);
- faturamento por km;
- **custo/km** = combustível + manutenção e pneus + custos fixos rateados por km, com **nível de confiança** (baixo, médio ou alto) e a base usada:
  - **estimado:** informado pelo motorista, confiança baixa;
  - **observado:** a partir de abastecimentos, manutenções e KM reais, janela de 90 dias, confiança por componente (regras da revisão da 0007);
  - **personalizado:** ajuste manual, mantendo o observado visível;
- **resultado econômico** = faturamento − KM × custo/km;
- **resultado de caixa** = faturamento − o que foi efetivamente pago;
- **ganho real por hora** = resultado econômico ÷ horas online (a tela mostra qual tempo foi usado; sem horas, "sem dado");
- médias diária, semanal e mensal.

O valor das corridas e ofertas da Uber já vem sem a taxa da plataforma. Não descontar a taxa de novo.

Não usar `float` para valores monetários. Usar **`Decimal`** no PostgreSQL/Prisma e uma representação decimal segura no TypeScript.

PRECISA DEFINIR (ADR): precisão das colunas (o custo/km precisa de mais casas que o centavo), regra de arredondamento (a 0007 usava meio longe do zero, só no fim) e biblioteca decimal no TypeScript. O `libs/metrics` atual usa inteiros (centavos) e precisa ser adaptado.

---

# 14. MÓDULO EXPENSES

Categorias: combustível, manutenção (inclusive preventiva), lavagem, pneus, óleo, seguro, IPVA, financiamento, estacionamento, pedágio, alimentação, outros.

```text
id
driverId
vehicleId
category
amount
description
date
odometer
metadata
createdAt
updatedAt
```

- Combustível: preencher 2 de 3 campos (litros, preço, total) e marcar tanque cheio.
- KM inicial e final do dia.
- Categorias extensíveis.

---

# 15. MÓDULO GOALS

Metas diária, semanal e mensal (faturamento, resultado e horas). Mostrar meta, realizado, percentual, restante e projeção.

---

# 16. MÓDULO SUBSCRIPTIONS

Módulo crítico:

- plano (mensal ou anual) e produto da Google Play;
- assinatura, status, purchase token;
- início, expiração, cancelamento, renovação;
- teste grátis de 7 dias;
- período de carência e estado de cobrança, quando disponíveis;
- histórico.

```text
ACTIVE
EXPIRED
CANCELED
PAUSED
GRACE_PERIOD
ON_HOLD
PENDING
UNKNOWN
```

O backend valida a assinatura pelas APIs oficiais da Google Play, isoladas em `GooglePlaySubscriptionService`. Não espalhar chamadas da Google Play pelo código.

Enviar o hash do `userId` em `obfuscatedAccountId`.

---

# 17. GOOGLE PLAY BILLING

O app:

1. consulta os produtos;
2. apresenta os planos (Mensal selecionado por padrão);
3. inicia a compra;
4. obtém o resultado e o purchase token;
5. envia o token ao backend;
6. o backend valida e registra/atualiza a assinatura;
7. o app atualiza o estado da conta.

O app não usa o retorno local da compra como única fonte de autorização. Oferecer "Restaurar compra" e "Gerenciar na Google Play".

Preparar as notificações de desenvolvedor em tempo real (RTDN) da Google Play quando necessário. Seguir sempre a documentação oficial mais atual.

---

# 18. MÓDULO LICENSES

Camada de autorização independente da assinatura:

```text
User → Subscription → License → Features
```

```text
TRIAL
PREMIUM
EXPIRED
```

PRECISA VALIDAR: o rascunho previa `FREE`, mas o produto é só pago (só teste grátis).

Entitlements, por exemplo:

```text
RIDE_ANALYSIS
ADVANCED_DASHBOARD
GOALS
EXPENSES
OVERLAY
HISTORY
ANALYTICS
```

PRECISA VALIDAR: limite de aparelhos por conta (citado no rascunho, sem número definido).

---

# 19. MÓDULO NOTIFICATIONS

Arquitetura para notificações locais e push, avisos de assinatura e de meta, lembretes (incluindo manutenção preventiva) e alertas financeiros.

Não implementar infraestrutura complexa de push se não for necessária no MVP. Criar abstrações para o futuro.

---

# 20. IMPORTAÇÃO DO RELATÓRIO SEMANAL DA UBER (MVP 1)

Especificação completa em `docs/parser-uber-relatorio-semanal.md` (copiar do repositório `Pilotei`).

- O motorista baixa o PDF em Ganhos › Relatórios semanais do portal (CONFIRMADO). O botão "Abrir portal da Uber" abre `drivers.uber.com` em Custom Tab; a URL vem de configuração remota.
- O PDF entra pelo seletor de arquivos, **é lido no celular e nunca sai do aparelho**. O app envia ao backend só as corridas lidas.
- Parser em TypeScript puro com o pdf.js (`getTextContent()`), separando colunas pela coordenada X. PRECISA VALIDAR o pdf.js no Hermes; se falhar, só a extração vira módulo nativo com PdfBox-Android.
- Chave contra duplicatas: hash de (UBER, data e hora do evento, categoria, valor).
- Validações obrigatórias: soma das transações = total "Seus ganhos" e saldo inicial + ganhos = saldo final. Se falhar, avisar e não importar em silêncio.
- Prévia antes de salvar, com o selo "Total conferido". Cada importação é um lote que pode ser desfeito.
- Dados pessoais do PDF (nome, telefone, e-mail): descartar.
- O PDF semanal substitui os dias manuais da Uber, sem contar nada em dobro.
- Testes golden com PDF sintético gerado por script; o PDF real fica fora do git.
- A **API da Uber está descartada** (os termos proíbem agregar dados com concorrentes e usá-los em análise).
- 99: importação "pós-MVP", desativada; lançamento manual.

Entidade adicional necessária (PROPOSTA): `ImportBatch` (origem, hash do arquivo, período, contagens, erros).

---

# 21. ACESSIBILIDADE ANDROID — COPILOTO (MVP 2)

Uma das partes mais importantes e mais delicadas do projeto. Fica no **MVP 2**, em **Kotlin**, como Expo Module.

**Antes do MVP 2:** um app de teste para confirmar, em 5 a 10 ofertas reais, se o app do motorista da Uber mostra os valores na árvore de acessibilidade. Se não mostrar, usar `takeScreenshot()` (Android 11+) com OCR; MediaProjection fica como alternativa.

O serviço deve:

- identificar o app em primeiro plano;
- detectar mudanças relevantes na interface;
- analisar os nós de acessibilidade disponíveis;
- extrair só as informações necessárias e normalizá-las;
- enviar eventos para o analisador da oferta.

```text
AccessibilityService
        ↓
PlatformDetector
        ↓
AccessibilityParser
        ↓
RideOffer
        ↓
RideAnalyzer
        ↓
OverlayController
```

O Copiloto liga e desliga em Configurações. A permissão só é pedida ao ativar.

Riscos: declaração de acessibilidade na Google Play, FLAG_SECURE e o modo de Proteção Avançada do Android 16.

---

# 22. SUPORTE A PLATAFORMAS NO COPILOTO

```kotlin
interface RidePlatformParser {
    fun supports(packageName: String): Boolean

    fun parse(
        accessibilityEvent: AccessibilityEvent
    ): RideOffer?
}
```

Implementar primeiro **`UberParser`**. `NineNineParser` e `InDriveParser` vêm depois, cada um com seu próprio teste de árvore de acessibilidade.

```text
accessibility/
├── RideAccessibilityService.kt
├── platform/
│   ├── RidePlatformParser.kt
│   ├── UberParser.kt
│   ├── NineNineParser.kt     # depois
│   └── InDriveParser.kt      # depois
├── analyzer/
└── overlay/
```

Nunca assumir que a interface dos apps externos ficará estável. Criar testes para cada parser.

---

# 23. OVERLAY (MVP 2)

Overlay nativo, só quando o sistema permitir. Ele deve:

- aparecer só sobre os apps autorizados;
- ser pequeno, consumir poucos recursos, poder ser ocultado e reposicionado;
- respeitar o lifecycle;
- não bloquear o app de transporte;
- **não ter botões de aceitar ou recusar**: o app não aceita corridas.

Exemplo, com semáforo e voz:

```text
R$ 18,50
12,4 km
32 min

R$ 1,49/km
R$ 34,68/h

● BOA
```

A regra de classificação é configurável. Nenhuma regra financeira fixa na UI.

---

# 24. MOTOR DE ANÁLISE DE CORRIDAS

Componente independente: `RideAnalyzer`.

Entrada: valor, distância, tempo, custo/km do motorista, configurações e metas.

Saída:

```text
grossAmount
netAmount
estimatedFuelCost
estimatedMaintenanceCost
estimatedDepreciation
estimatedProfit
profitPerKm
profitPerHour
rating
```

```text
EXCELLENT
GOOD
REGULAR
BAD
```

Limites configuráveis (R$/km mínimo, R$/hora mínimo, resultado mínimo). O algoritmo é substituível. O valor da oferta da Uber já vem sem a taxa; não descontar de novo.

PRECISA DEFINIR: onde roda a análise no Copiloto. A oferta precisa de resposta imediata na tela, e o app só tem cache de leitura; a proposta é calcular no aparelho com o custo/km já calculado pelo servidor.

---

# 25. DASHBOARD

### Hoje

- faturamento bruto e líquido;
- despesas;
- resultado econômico estimado;
- km rodados e horas online;
- ganho real por hora e faturamento por km;
- meta diária.

### Semana

- faturamento, resultado, despesas, corridas, km e horas.

### Mês

- receita, despesas, resultado, média, evolução e comparação com o mês anterior.

Também: ganhos por plataforma (só em texto) e próxima manutenção.

No primeiro acesso, o dashboard mostra "Complete seu veículo e custos": conta, veículo, custos fixos e valores mínimos.

Usar gráficos só quando ajudarem a entender.

---

# 26. NAVEGAÇÃO

```text
Dashboard
Corridas
Ganhos
Despesas
Metas
Veículos
Análises
Configurações
Assinatura
```

PRECISA DEFINIR: quais seções ficam no menu inferior e quais ficam em menus secundários. O protótipo em `design/` (Início, Financeiro e Relatórios) precisa ser revisado para essa navegação.

Visual: tema escuro e claro, seguindo o celular; fonte Inter; verde de destaque `#00C896` com texto escuro nos botões (tokens no `CLAUDE.md` do repositório `Pilotei`).

Interface moderna e profissional, com legibilidade para uso durante o trabalho e sem excesso de informação.

---

# 27. CONECTIVIDADE

- **Internet obrigatória para lançar.** Sem sinal, o app só consulta os últimos dados carregados.
- **Não há sincronização offline**, fila local nem resolução de conflitos.
- O cache local é só de leitura.
- O backend é a fonte da verdade.
- Envios devem ser idempotentes (chave de idempotência por requisição, PROPOSTA) para não duplicar registros quando o motorista reenvia após falha de rede.

---

# 28. BANCO DE DADOS

PostgreSQL e Prisma 7. Schema completo, com migrations. Nunca depender de alterações manuais no banco.

Entidades mínimas:

```text
User
Driver
Vehicle
Ride
Earning
Expense
Goal
Subscription
License
Notification
RefreshToken
OtpCode         # PROPOSTA (login sem senha)
ImportBatch     # PROPOSTA (importação do PDF)
```

Usar UUID, timestamps, índices, foreign keys, constraints, unique constraints, `Decimal` e enums quando apropriado.

Datas em UTC, exibidas no fuso America/Sao_Paulo.

---

# 29. MULTI-TENANCY

```text
User
   ↓
Driver
   ↓
Vehicle
   ↓
Rides / Expenses / Goals / Earnings
```

Um usuário nunca acessa dados de outro. Toda consulta tem controle de ownership, validado no backend.

---

# 30. SEGURANÇA

- JWT e refresh token com rotação;
- OTP com hash, limite de tentativas e rate limiting;
- validação de DTO e sanitização;
- CORS e Helmet;
- logs sem dados pessoais;
- autorização por usuário e controle de acesso;
- proteção contra IDOR e mass assignment;
- secrets em variáveis de ambiente.

Nunca colocar no código-fonte: `JWT_SECRET`, credenciais do Google, chave do Resend, senha do banco ou chaves privadas.

---

# 31. LGPD

- Termos de Uso e Política de Privacidade, com o aviso de não afiliação;
- consentimento quando necessário;
- exclusão de conta e de dados;
- exportação de dados;
- minimização de dados (o PDF não sai do aparelho; dados pessoais do PDF são descartados);
- documentação dos dados coletados.

Pedir só as permissões necessárias. Explicar ao motorista por que a acessibilidade é necessária. Não coletar conteúdo desnecessário da tela.

---

# 32. ACESSIBILIDADE E GOOGLE PLAY

O AccessibilityService serve só às funcionalidades legítimas do app.

Não implementar:

- captura indiscriminada da tela;
- espionagem;
- coleta de senhas ou de dados bancários;
- interceptação de mensagens;
- coleta sem finalidade;
- comportamento oculto.

O motorista ativa o serviço explicitamente. Antes da publicação, revisar a implementação contra as políticas atuais da Google Play (AccessibilityService, permissões, overlays, dados do usuário e billing).

---

# 33. API

API REST, base `/api/v1`, com Swagger.

```text
POST /auth/google
POST /auth/otp/request
POST /auth/otp/verify
POST /auth/refresh
POST /auth/logout

GET    /users/me
DELETE /users/me

GET   /drivers/me
PATCH /drivers/me

GET    /vehicles
POST   /vehicles
PATCH  /vehicles/:id
DELETE /vehicles/:id

GET  /rides
POST /rides
GET  /rides/:id

POST   /imports/uber-weekly
DELETE /imports/:id

GET /earnings/summary
GET /earnings/daily
GET /earnings/weekly
GET /earnings/monthly

GET    /expenses
POST   /expenses
PATCH  /expenses/:id
DELETE /expenses/:id

GET   /goals
POST  /goals
PATCH /goals/:id

GET  /subscriptions/me
POST /subscriptions/google-play/validate

GET /licenses/me
```

Rotas de exemplo (PROPOSTA), a detalhar em `docs/api.md`.

---

# 34. OBSERVABILIDADE

- logs estruturados;
- request ID;
- tratamento global de exceções;
- health check e readiness;
- métricas básicas.

```text
GET /health
GET /health/ready
```

Não adicionar infraestrutura pesada sem necessidade.

---

# 35. DOCKER

Dockerfile do backend e `docker-compose.yml` com:

```text
pilotei-backend
postgres
```

Volumes persistentes e healthchecks. O PostgreSQL não fica exposto publicamente.

Desenvolvimento no WSL2 (Ubuntu 24), com Docker Engine.

---

# 36. VPS

Deploy na **Hetzner CPX22**:

```text
Internet
   ↓
Reverse Proxy (HTTPS)
   ↓
Pilotei Backend
   ↓
PostgreSQL
```

HTTPS obrigatório. O PostgreSQL fica privado; a porta 5432 não é exposta à internet.

---

# 37. ENVIRONMENT

`.env.example`:

```env
NODE_ENV=development

DATABASE_URL=

JWT_SECRET=
JWT_REFRESH_SECRET=

GOOGLE_OAUTH_CLIENT_ID=

RESEND_API_KEY=
EMAIL_FROM=

GOOGLE_PLAY_PACKAGE_NAME=
GOOGLE_PLAY_SERVICE_ACCOUNT=

CORS_ORIGINS=

LOG_LEVEL=
```

Nunca colocar valores reais no Git.

---

# 38. TESTES BACKEND

### Unit Tests

- custo/km e confiança;
- resultado econômico, resultado de caixa e ganho real por hora;
- metas;
- autorização;
- assinatura e licença.

### Integration Tests

- autenticação (Google e OTP);
- usuários, veículos, corridas, despesas;
- importação (duplicatas e desfazer lote);
- assinatura e licença.

### E2E

Testar os principais fluxos da API.

---

# 39. TESTES DO APP

- unitários (TypeScript);
- componentes e telas;
- leitor do PDF da Uber, com testes golden;
- cache de leitura;
- no MVP 2: parsers de acessibilidade (Kotlin), `RideAnalyzer` e regras de classificação.

Não depender só de testes manuais.

---

# 40. DOCUMENTAÇÃO

```text
docs/
├── architecture.md
├── architecture-decisions/
├── database.md
├── api.md
├── app.md
├── uber-pdf-import.md
├── accessibility.md
├── subscriptions.md
├── google-play.md
├── security.md
├── lgpd.md
├── deployment.md
├── development.md
├── testing.md
├── roadmap.md
└── progress.md
```

E também `README.md` e `AGENTS.md`. O `AGENTS.md` contém as regras arquiteturais que o Claude Code deve respeitar em futuras sessões.

---

# 41. CI/CD

GitHub Actions:

```text
pull request
    ↓
lint
    ↓
unit tests
    ↓
integration tests
    ↓
build
```

- Backend: instalar, lint, test e build.
- App: lint, test e `expo prebuild` + build Android de debug (ou EAS Build, conforme a FASE 0).

Preparar a geração do **AAB** para a Google Play. Não publicar automaticamente na Play Store no primeiro momento.

---

# 42. PADRÕES DE CÓDIGO

Aplicar SOLID, Clean Code, inversão de dependência, composição, baixo acoplamento e alta coesão.

Evitar God Classes, God Services, controllers gigantes, repositories com regra de negócio, lógica duplicada, constantes espalhadas, strings mágicas e números mágicos.

---

# 43. REGRA IMPORTANTE SOBRE REGRAS DE NEGÓCIO

Não colocar regras financeiras na UI.

Errado, dentro da tela:

```text
if (price / km > 2)
```

Correto: o `RideAnalyzer` (no Copiloto) ou o módulo de métricas (no backend) contém a regra; o app só apresenta o resultado.

---

# 44. FLUXO DE PRIMEIRO ACESSO

```text
Abertura
 ↓
Boas-vindas (Criar conta / Já tenho conta)
 ↓
Login: Google ou e-mail + código
 ↓
Configuração do motorista
 ↓
Cadastro do veículo
 ↓
Combustível e custos fixos
 ↓
Valores mínimos e metas
 ↓
Dashboard
```

A configuração da acessibilidade e do overlay acontece só quando o motorista ativa o Copiloto (MVP 2).

---

# 45. FLUXO DE ASSINATURA

```text
Usuário
 ↓
Cadastro
 ↓
Teste grátis de 7 dias (só quem nunca assinou)
 ↓
Tela de assinatura (Mensal R$ 10,99 / Anual R$ 97,99)
 ↓
Google Play Billing
 ↓
Purchase Token
 ↓
Backend
 ↓
Google Play Developer API
 ↓
Validação
 ↓
Subscription
 ↓
License
 ↓
Recursos liberados
```

Estados exibidos: teste, ativo, cancelado e expirado.

---

# 46. DASHBOARD FINANCEIRO

### Receita

Total bruto, total líquido, média diária e média por corrida.

### Despesas

Combustível, manutenção, pneus, seguro, IPVA, estacionamento, pedágio e outros.

### Resultados

```text
Resultado econômico = Faturamento − KM × custo/km
Resultado de caixa  = Faturamento − pagamentos efetivos
```

### Indicadores

```text
R$/km (faturamento)
custo/km (com confiança)
resultado/km
ganho real por hora
```

### Metas

Meta, realizado, percentual, restante e projeção.

---

# 47. ANÁLISES

Melhor e pior dia, melhor e pior horário, médias por corrida, por hora e por km, custo/km, resultado por km e por hora, evolução mensal.

Preparar a estrutura para o MVP 3 (região, categoria, KM vazio, tempo ocioso e metas dinâmicas).

---

# 48. PRIVACIDADE NO COPILOTO

- processar só os dados necessários;
- não armazenar screenshots;
- não capturar senhas nem dados bancários;
- não coletar conteúdo indiscriminadamente;
- não enviar conteúdo bruto da tela ao backend;
- normalizar no aparelho.

```text
Tela externa → Parser → RideOffer estruturada → Analyzer
```

em vez de enviar o conteúdo da tela ao servidor.

---

# 49. UX

O app é usado enquanto o motorista trabalha:

- textos grandes e alto contraste;
- poucos elementos e ações rápidas;
- feedback visual claro;
- telas simples;
- nada de digitação durante a corrida.

O motorista deve entender rapidamente: **"Essa corrida vale a pena?"**

---

# 50. MVP

### MVP 1 — Controle

**App:** login (Google/OTP), dashboard, veículo, despesas (inclusive manutenção preventiva), custos fixos, metas, corridas e dias manuais, importação do PDF semanal da Uber, cálculo financeiro, assinatura.

**Backend:** Auth, Users, Drivers, Vehicles, Rides, Imports, Earnings, Expenses, Goals, Metrics, Subscriptions, Licenses, Notifications, Prisma, PostgreSQL, Swagger, Docker e testes.

### MVP 2 — Copiloto

App de teste de acessibilidade com a Uber; AccessibilityService, `UberParser`, `RideAnalyzer`, overlay com semáforo e voz e comparação oferta × resultado. Depois, 99 e inDrive.

### MVP 3 — Inteligência

Análises por horário, região e categoria, KM vazio, tempo ocioso e metas dinâmicas.

### Pós-MVP

iOS (sem o Copiloto), importação da 99, histórico pelo "Baixar seus dados" da Uber e o Pilotei APP (passageiro).

---

# 51. ROADMAP DE IMPLEMENTAÇÃO

Não implementar tudo de uma vez. Executar nesta ordem:

## FASE 0 — Discovery

Criar `AGENTS.md`, `docs/architecture.md`, `docs/roadmap.md` e as primeiras ADRs em `docs/architecture-decisions/`, incluindo as decisões desta revisão e as pendências marcadas como PRECISA DEFINIR. Não escrever funcionalidades.

## FASE 1 — Monorepo

Estrutura `apps/pilotei-driver`, `apps/pilotei-backend`, `packages/`, `docs`, `docker`, `scripts`. Git, lint, format e CI.

## FASE 2 — Backend Base

NestJS, Prisma 7, PostgreSQL, configuração, health, Swagger e estrutura modular. Migrar o `libs/metrics` do `Pilotei-Backend`.

## FASE 3 — Auth

Google, OTP pelo Resend, refresh, logout e proteção.

## FASE 4 — Domínio

Drivers, Vehicles, Rides, Expenses, Earnings, Goals e Metrics.

## FASE 5 — App Base

Projeto Expo (development build), navegação, arquitetura, cache de leitura, cliente HTTP e autenticação.

## FASE 6 — Importação do PDF da Uber

Leitor TypeScript + pdf.js com testes golden (validar no Hermes) e endpoint de importação com lote e duplicatas.

## FASE 7 — Dashboard

Dashboard e financeiro.

## FASE 8 — Google Play

Billing, planos, teste grátis, purchase token, validação, License e entitlements.

## FASE 9 — Segurança

Auditoria de autenticação, autorização, ownership, secrets, logs, API, banco e app.

## FASE 10 — Testes

Executar todos os testes e corrigir as falhas.

## FASE 11 — Release do MVP 1

Android AAB, imagem Docker do backend e documentação de deploy na Hetzner.

## FASE 12 — Copiloto (MVP 2)

App de teste de acessibilidade com a Uber; depois AccessibilityService, `UberParser`, `RideOffer`, `RideAnalyzer` e overlay.

---

# 52. CRITÉRIO PARA AVANÇAR DE FASE

Ao terminar cada fase:

1. executar testes;
2. executar lint;
3. executar build;
4. revisar arquitetura;
5. revisar segurança;
6. revisar documentação;
7. verificar os arquivos modificados;
8. corrigir problemas;
9. registrar o progresso em `docs/progress.md`.

```text
FASE 0 - DONE
FASE 1 - DONE
FASE 2 - IN PROGRESS
...
```

Somente depois avançar.

---

# 53. FLUXO DE TRABALHO

- Cada fase ou feature tem uma branch numerada a partir de `develop` (`001-...`, `002-...`).
- Ao final, um PR para `develop` descreve o que foi implementado e o que deve ser testado.
- Não mexer no `main` sem aprovação. Não commitar sem autorização. Commits sem co-autoria.
- Nada de código antes de o plano da etapa ser aprovado.

---

# 54. COMPORTAMENTO ESPERADO DO CLAUDE CODE

Agir como Software Architect, Backend Engineer, Mobile Engineer (React Native e Android), QA Engineer, DevOps Engineer e Security Engineer. Não só gerar código.

Antes de modificar grandes partes do projeto:

1. analisar o código existente;
2. entender a arquitetura;
3. verificar dependências;
4. identificar impactos;
5. implementar;
6. testar;
7. revisar.

Nunca apagar código funcional sem motivo. Nunca substituir a arquitetura sem justificar. Nunca adicionar dependências sem verificar se são necessárias.

---

# 55. DOCUMENTAÇÃO DE DECISÕES

Toda decisão arquitetural importante vira ADR em `docs/architecture-decisions/`:

```text
ADR-001-monorepo.md
ADR-002-monolito-modular.md
ADR-003-react-native-expo.md
ADR-004-autenticacao-sem-senha.md
ADR-005-internet-obrigatoria.md
ADR-006-dinheiro-decimal.md
ADR-007-google-play-billing.md
ADR-008-importacao-pdf-uber.md
ADR-009-accessibility-service.md
ADR-010-hospedagem-hetzner.md
```

As decisões 0001 a 0008 do repositório `Pilotei` são o histórico e devem ser citadas nas ADRs que as substituem.

---

# 56. REGRA PARA DEPENDÊNCIAS

Antes de adicionar uma biblioteca:

1. verificar se a funcionalidade já existe no React Native, no Expo ou no Android;
2. verificar a compatibilidade (Hermes, nova arquitetura, versão do Expo SDK);
3. verificar a manutenção;
4. verificar a licença;
5. avaliar o impacto no tamanho do app;
6. adicionar só se necessário.

---

# 57. REGRA PARA DOCUMENTAÇÃO EXTERNA

Para APIs externas, principalmente Google Play Billing, Google Play Developer API, AccessibilityService, overlay do Android, Expo e políticas da Google Play: consultar sempre a documentação oficial mais atual antes de implementar.

Não inventar APIs. Não usar métodos depreciados quando houver alternativa oficial. Separar o que é CONFIRMADO, PROPOSTA e PRECISA VALIDAR.

---

# 58. REGRA PARA ACESSIBILIDADE

Antes de considerar o Copiloto pronto, verificar permissões, lifecycle, consumo de bateria, memória, detecção do app, tratamento de eventos, parser, mudanças de UI, segurança e a política da Google Play.

Criar uma camada de compatibilidade para que mudanças na UI dos apps externos não contaminem o restante do sistema.

---

# 59. RESULTADO FINAL ESPERADO

```text
Pilotei-Driver (monorepo)
│
├── App React Native + Expo
│   ├── Leitor do PDF da Uber
│   ├── Cache de leitura
│   ├── Google Play Billing
│   └── Copiloto (Kotlin, MVP 2)
│       ├── AccessibilityService
│       ├── Overlay
│       └── RideAnalyzer
│
├── Backend NestJS
│   ├── Monólito modular
│   ├── Prisma 7
│   ├── PostgreSQL
│   ├── JWT + Google + OTP
│   ├── Métricas
│   ├── Subscriptions
│   ├── Licenses
│   └── REST API
│
├── Docker
├── CI/CD
├── Tests
├── Documentation
└── Deployment (Hetzner)
```

O resultado deve ser um projeto realmente executável, não um protótipo ou código ilustrativo.

---

# 60. REGRA FINAL

Não diga apenas que uma etapa foi implementada. Depois de implementar, execute os comandos de validação (lint, test e build do backend e do app).

Se houver erro:

1. identificar;
2. corrigir;
3. executar de novo;
4. só considerar concluído quando estiver funcionando.

Prioridade, nesta ordem:

```text
Correto
Seguro
Testável
Manutenível
Simples
Escalável
```

Comece pela **FASE 0 — Discovery**: analise o ambiente atual e crie `AGENTS.md`, `docs/architecture.md`, `docs/roadmap.md` e as ADRs iniciais. Não implemente todas as fases de uma vez.

Ao finalizar cada fase, apresente um resumo do que foi criado, dos testes executados, dos problemas encontrados e da próxima fase.
