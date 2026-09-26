# Pilotei — Contexto do projeto (handoff para o Claude Code)

> Repositório: `https://github.com/MarlosAraujo/Pilotei.git`. Especificações em `docs/`, protótipo das telas em `design/`.
> Origem: conversa de planejamento no Claude (Cowork), de 24 a 26/09/2026. O produto se chamava "Destival" e, nos primeiros protótipos, "DriverCost".

## Como trabalhar comigo
- Respostas breves e diretas. Pesquisas curtas, focadas no resultado.
- Trabalhar sempre em branch próprio. Não mexer no `main` sem aprovação.
- Cada feature tem uma branch numerada em sequência, criada a partir de `develop`: `001-setup/estrutura-inicial`, `002-setup-stack`, `003-...`.
- Ao final de cada feature, abrir um PR para `develop` com descrição detalhada do que foi implementado e do que deve ser testado.
- Nada de código até eu aprovar a etapa. A fase atual é definir cada ponto do app e do backend. Só depois vem a criação de código.
- Ao citar Uber, Google Play, Android etc., separar **CONFIRMADO**, **PROPOSTA** e **PRECISA VALIDAR**. Não inventar APIs.

## O produto
App Android para motoristas de aplicativo (Uber, 99, inDrive e corridas particulares). Ele junta:
**gestão do veículo + custos reais + ganhos por plataforma + análise de ofertas + inteligência histórica.**

A ideia central é um ciclo de dados: o app começa com o custo **estimado**, informado pelo motorista. Com abastecimentos, manutenções e KM reais, passa a usar o custo **observado** e depois o **personalizado**, sempre com **nível de confiança** e explicação da base usada.

Não confundir nunca: receita ≠ lucro; R$/km ≠ lucro/km; oferta ≠ corrida; estimativa ≠ real; resultado de caixa ≠ resultado econômico.

## Roadmap
- **MVP 1 — Controle:** veículo, abastecimento, manutenção (inclusive preventiva), custos fixos, lançamentos manuais, importação do relatório semanal da Uber em PDF, dashboard, financeiro, metas e assinatura.
- **MVP 2 — Copiloto:** leitura da oferta na tela, semáforo, voz e comparação oferta × resultado.
- **MVP 3 — Inteligência:** análises por horário, região e categoria, KM vazio, tempo ocioso e metas dinâmicas.
- **Pós-MVP:** importação da 99 e importação do histórico pelo "Baixar seus dados" da Uber (ZIP com CSV).

## Decisões tomadas

### Nome e marca
- Nome: **Pilotei**. No logo, "Pilo" na cor do texto e "tei" em verde. Frase atual: "Saiba quanto vale sua corrida."
- PRECISA VALIDAR: disponibilidade do nome no INPI (classes 9 e 42), na Google Play, do domínio e das redes sociais.
- Marcas de terceiros (Uber, 99, inDrive) aparecem **só como palavra**, sem logo, ícone, selo ou cores de marca.
- Aviso obrigatório: "O Pilotei não é afiliado à Uber, 99 ou inDrive."

### Visual
- Tema escuro e claro, seguindo o celular. Fonte Inter. Verde de destaque `#00C896`, com texto escuro nos botões.
- Tokens usados no protótipo:
  - escuro: bg `#0B1411`, surface `#12201C`, surface2 `#182B26`, texto `#EEF8F4`, secundário `#93A8A0`, linha `#243A33`, verde `#19D9A0`, aviso `#F5B544`
  - claro: bg `#F4F7F6`, surface `#FFFFFF`, surface2 `#EAF2EF`, texto `#0F1E1A`, secundário `#56665F`, linha `#D9E3DF`, verde `#00C896`, verde para texto `#00755A`, aviso `#9A6400`

### Monetização (Google Play Billing)
- Só pago, com **7 dias de teste grátis** (apenas para quem nunca assinou).
- **Mensal: R$ 10,99** (definido). **Anual: R$ 99,90** (sugestão, ainda a confirmar), equivalente a R$ 8,33/mês, 24% mais barato.
- Um produto de assinatura com dois planos (mensal e anual), cada um com a oferta de teste. Os preços exibidos vêm da Play (`ProductDetails`), nunca fixos no código.
- Assinatura expirada: consulta e exportação ficam liberadas; novos lançamentos exigem assinatura. A exclusão de conta fica no Perfil, como a Google Play exige.

### Importação de ganhos
- **Uber: relatório semanal em PDF.**
  - A tela Atividades do portal não exporta nada (CONFIRMADO). O PDF sai em Ganhos › Relatórios semanais (CONFIRMADO).
  - O botão "Abrir portal da Uber" abre `drivers.uber.com` no navegador ou em Custom Tab. Nada de WebView, e o app não toca no login.
  - O arquivo entra pelo seletor de arquivos do Android (Storage Access Framework, sem permissão de armazenamento).
- **Digitação manual ("Adicionar dia")** para todas as plataformas: data, plataforma, ganhos, KM, horas online e número de corridas. Sempre disponível.
- **99:** o botão de importação aparece desativado com a etiqueta "Importação pós-MVP". No MVP, a 99 é lançada à mão.
- **A API da Uber foi descartada.** Os Termos da API (março de 2026, seção III.G) proíbem:
  - (e) mostrar a Uber junto com concorrentes;
  - (f) agregar dados da Uber com dados de concorrentes;
  - (g) analisar ou coletar em massa os dados da Uber, inclusive preços;
  - (h) usar dados da Uber em IA, recomendação ou decisão automatizada;
  - usar esses dados em app que também suporte concorrentes.

  Isso é incompatível com o Pilotei. Esses termos só valem para quem usa a API, então o PDF baixado pelo motorista e a digitação manual não são afetados.
- **Risco aceito pelo responsável do produto:** seguir com o PDF e com o Copiloto lendo a tela, sem analisar o contrato do motorista com a Uber.
- **Arquitetura:** interface `EarningsSource` com `FileSource` e `ManualSource`. Plataformas e formatos novos são ligados por feature flag remota.
- A URL do portal fica em configuração remota.

### Leitor do relatório semanal da Uber (PDF)
Especificação completa em `docs/parser-uber-relatorio-semanal.md`. Resumo:

- **Semana:** vai de segunda às 04:00 até a segunda seguinte às 04:00. O ano vem do cabeçalho, porque as linhas não trazem o ano.
- **Transações:** cada uma ocupa duas linhas.
  - Linha 1: processado (data) | evento | seus ganhos | saldo
  - Linha 2: hora do processamento | data e hora do evento | saldo acumulado
- **Mapeamento:**
  - "Uber X" → `Trip.category = "UberX"`
  - "Prioridade" → `Trip.category = "UberX Prioridade"`
  - "Promoção - ..." → `Income(type=BONUS)`, não é corrida
  - linha de R$ 0,00 → ajuste, vinculado ao evento de mesma data e hora
- **Tempos:** a data e hora do evento vira `started_at` (PRECISA VALIDAR se é o início ou o aceite). O processamento vira `finished_at`, aproximado.
- **O PDF não traz:** KM, ID da viagem, origem, destino nem taxa da Uber.
- **Chave contra duplicatas:** hash de (UBER, data e hora do evento, categoria, valor).
- **Validações obrigatórias:**
  - a soma das transações bate com o total "Seus ganhos" (no exemplo, 33 linhas = R$ 420,18 ✓);
  - saldo inicial + ganhos = saldo final;
  - se alguma falhar, mostrar aviso e não importar em silêncio.
- **Dados pessoais do PDF** (nome, telefone, e-mail): descartar.
- **Android:** usar PdfBox-Android (`com.tom_roush:pdfbox-android`) com as posições do texto, separando colunas pela coordenada X. Testes "golden" com PDFs anonimizados.

### Modelo de dados (plataformas)
- `Platform`: UBER, NINETY_NINE, INDRIVE, PRIVATE, OTHER. 99Pop é categoria da 99, não plataforma.
- `Trip`: platform, category, external_trip_id (ou hash), started_at, finished_at, duration_s, distance_m, gross_amount, net_amount, tips, platform_fee, status (concluída ou cancelada com taxa), source (FILE ou MANUAL), import_batch_id, metadata (JSON).
- `Income` (BONUS): promoções e bônus.
- `PlatformDaySummary`: dia lançado à mão (data, platform, net_amount, km, online_minutes, trips_count). Evita criar corridas inventadas.
- `ImportBatch`: origem, hash do arquivo, período, contagens e erros. Permite desfazer uma importação.
- `PlatformConnection`: platform, method, status, last_import_at, enabled.
- Dinheiro sempre em **centavos (Long)**. Datas em UTC, exibidas no fuso America/Sao_Paulo.
- O Dashboard soma Trip, Income e PlatformDaySummary sem contar nada em dobro. O PDF semanal substitui os dias manuais da Uber.

### Copiloto (MVP 2)
- **Captura:** o AccessibilityService serve de gatilho e lê a árvore de elementos da tela. Se a árvore não trouxer os valores, usar `takeScreenshot()` (Android 11+) com OCR. MediaProjection fica como alternativa.
- **Antes do MVP 2:** um app de teste para confirmar, em 5 a 10 ofertas reais, se o app do motorista da Uber mostra os valores na árvore de acessibilidade.
- **Riscos:** declaração de acessibilidade na Google Play, FLAG_SECURE e o modo de Proteção Avançada do Android 16.
- O Copiloto liga e desliga em Configurações. A permissão só é pedida ao ativar.
- A análise aparece por cima do app da Uber, com semáforo e voz, **sem botões de aceitar ou recusar**: o app não aceita corridas.
- O valor da oferta da Uber já vem sem a taxa da plataforma. Não descontar a taxa de novo.

### Métricas
- **Custo/km** = combustível + manutenção e pneus + custos fixos rateados por km, com nível de confiança (baixo, médio ou alto) e a base de dados mostrada.
- **Resultado econômico** = faturamento − KM × custo/km. **Resultado de caixa** = faturamento − o que foi efetivamente pago.
- **Ganho real por hora** = resultado econômico ÷ **horas online**. A tela deixa explícito qual tempo foi usado.
- **Números de exemplo nas telas:** faturamento R$ 402,60; 200 km × R$ 0,98 = R$ 196,00; resultado R$ 206,60; 7h30 online; R$ 27,55/h.

## Telas (protótipo no canvas "Pilotei — Telas do app", no claude.ai)
Link: https://claude.ai/artifact/8Ndkpkogppxa8YmM6LCXzM (privado; só abre na sua conta). Cópia local em `design/`.

1. **Acesso:** Abertura → Boas-vindas (Criar conta / Já tenho conta) → Criar conta (Google ou e-mail) / Entrar.
2. **Uso diário:**
   - **Dashboard do primeiro acesso:** "Complete seu veículo e custos", com lista de passos: conta, veículo, custos fixos e valores mínimos.
   - **Dashboard gerencial:**
     - botões de Perfil e Configurações no topo;
     - Hoje / Semana / Mês;
     - resultado econômico estimado;
     - ganho real por hora e faturamento por km;
     - ganhos por plataforma, só em texto;
     - próxima manutenção.
   - **Financeiro:**
     - botões Importar ganhos e Adicionar dia;
     - ganhos por plataforma;
     - custos do dia: combustível, KM do dia e outra despesa;
     - resultado de caixa e resultado econômico.
   - **Importar ganhos:**
     - Uber: portal + importar PDF;
     - 99: importação desativada, digitação manual ativa;
     - inDrive e Particulares: digitação manual.
   - **Importar relatório:** prévia antes de salvar, com semana, corridas, promoções, ajustes, duplicadas e o selo "Total conferido".
   - **Adicionar dia (manual)** e **Novo lançamento de custo:** combustível (preenche 2 de 3 campos, tanque cheio), KM inicial e final do dia, manutenção e despesa.
   - **Menu inferior:** Início, Financeiro e Relatórios.
3. **Conta:**
   - **Configurações:**
     - Copiloto (liga/desliga);
     - seu custo estimado, custos fixos, valor mínimo por KM e por hora, metas;
     - veículo, voz, tema e backup.
   - **Perfil:** dados, status da assinatura, sair e excluir conta.
   - **Assinatura:** estados teste, ativo, cancelado e expirado; Mensal R$ 10,99 selecionado; Anual R$ 99,90; gerenciar na Google Play; restaurar compra.
4. **Detalhes:** Seu custo estimado (com confiança), Custos fixos, Metas e Veículo (uso pessoal sim/não).

## Arquitetura técnica (PROPOSTA, ainda não aprovada)
- **Base:** Kotlin, Jetpack Compose e Room, offline-first. Os dados ficam no celular e o backup é opcional.
- **Pagamento:** Google Play Billing.
- **Módulos:** `core-model`, `core-db`, `engine` (custos e métricas, sem dependência de Android e testável), `import` (EarningsSource e leitores de arquivo), `feature-*` (dashboard, financeiro, veículo, configurações, assinatura) e, no futuro, `copilot`.
- **Configuração remota** para feature flags e URLs dos portais.
- **Backend:** haverá um backend (decidido em 26/09/2026). Stack, responsabilidades (conta, assinatura, backup, sincronização, configuração remota) e hospedagem ainda PRECISAM SER DEFINIDOS, ponto a ponto, antes de qualquer código.

## Pendências
1. Confirmar o preço do plano anual.
2. Verificar a disponibilidade do nome Pilotei.
3. Confirmar se o portal da Uber gera o relatório da semana ainda em andamento.
4. Conseguir um PDF de exemplo da 99 (pós-MVP).
5. Fazer o app de teste de acessibilidade antes do MVP 2.
6. Revisar os Termos de Uso e a Política de Privacidade do Pilotei (LGPD), com o aviso de não afiliação.
7. Definir a arquitetura do backend (stack, responsabilidades e hospedagem).

## Próximos passos
1. ~~`001-setup/estrutura-inicial`: organizar `CLAUDE.md`, `docs/` e `design/` no repositório.~~
2. Definir, ponto a ponto, a stack do app e do backend (`002-setup-stack`), registrando as decisões em `docs/`.
3. Após aprovação: montar a estrutura do projeto Android (módulos acima) e começar pelo `engine` (custo/km, resultado e ganho real por hora), com testes unitários.
4. Em seguida: o leitor do PDF da Uber, com testes golden usando um PDF anonimizado.
