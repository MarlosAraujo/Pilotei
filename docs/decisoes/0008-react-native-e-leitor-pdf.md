# 0008 — Apps em React Native + Expo e leitor do PDF da Uber

- **Data:** 04/10/2026
- **Status:** Aprovado (04/10/2026)
- **Branch:** `007-react-native-e-leitor-pdf`

## Contexto
A `0002-stack-apps-e-backend.md` escolheu Kotlin nativo para os apps. Ao preparar o leitor do PDF da Uber, o primeiro código dos apps, o responsável pelo produto decidiu trocar para React Native: uma linguagem só (TypeScript) no app e no backend, e o iOS como opção no pós-MVP. A troca muda a forma de ler o PDF, o ambiente de desenvolvimento e algumas bibliotecas citadas nas decisões 0003, 0004 e 0005. O backend não muda.

## Decisão

### 1. Apps em React Native + Expo
- **React Native com Expo, em _development build_.** O Expo Go não é usado, porque não roda código nativo próprio (CONFIRMADO, `comparacao-stack.md`).
- **Linguagem:** TypeScript estrito, como no backend.
- **Código nativo** (acessibilidade e sobreposição do Copiloto, e o que mais faltar) entra por **Expo Modules API** em Kotlin, com **config plugins** para o manifesto (CONFIRMADO, `comparacao-stack.md`).
- Vale para os dois apps: **Pilotei Driver** e, no pós-MVP, **Pilotei APP**.
- Versões fixadas na branch de montagem do app, como na 0007.

### 2. Plataformas
- **MVP: só Android.**
- **Pós-MVP: iOS, sem o Copiloto.** O iOS não permite ler a tela de outro app nem desenhar por cima dele (CONFIRMADO, `comparacao-stack.md`). No iOS, a assinatura passa a ser pela App Store e exige uma conta Apple Developer. Decidir no pós-MVP.

### 3. Repositório e ambiente
- **Repositório próprio e privado:** `Pilotei/Pilotei-Driver`, na organização `Pilotei` do GitHub. Mesmo fluxo: nada no `main` sem aprovação, `develop` como base, branches numeradas (`001-...`) e PR para `develop`. As decisões continuam neste repositório.
- **Código no WSL**, como o backend. O Metro e os testes rodam no WSL.
- **Emulador no Windows** (Android Studio), ligado ao Metro pelo `adb`. A _development build_ é gerada com o Android SDK no WSL ou pelo EAS Build na nuvem. PRECISA VALIDAR qual das duas, na montagem do app.
- O app chega ao `gateway` em `http://10.0.2.2:7000` no emulador (PRECISA VALIDAR, como na 0004).

### 4. Leitor do relatório semanal da Uber
- **Parser em TypeScript puro.** Recebe a lista de textos com as posições (X, Y e página) e devolve as transações, o resumo e o resultado das validações. Não depende de React Native, e por isso roda e é testado no Node.
- **Extração com o pdf.js** (`pdfjs-dist`): o `getTextContent()` devolve cada trecho de texto com a matriz de posição, de onde saem X e Y (CONFIRMADO, API do pdf.js). As colunas são separadas pela coordenada X, como a especificação já previa.
- **PRECISA VALIDAR:** se o pdf.js roda no Hermes (motor JavaScript do React Native). Se não rodar, só a extração vira um módulo nativo com PdfBox-Android (Expo Module em Kotlin), entregando a mesma lista de textos com posições. O parser não muda.
- **O PDF continua no aparelho** (`0001-backend-fonte-da-verdade.md`): o app envia ao backend só as corridas já lidas.
- **Seleção do arquivo:** pelo seletor de documentos do sistema, sem permissão de armazenamento. PROPOSTA: `expo-document-picker`; PRECISA VALIDAR que ele usa o seletor do sistema no Android.
- **Portal da Uber:** aberto no navegador ou em Custom Tab, sem WebView. PROPOSTA: `expo-web-browser`.
- Regras de leitura, mapeamento e validações: sem mudança (`docs/parser-uber-relatorio-semanal.md`).

### 5. Testes golden do leitor
- **PDF sintético no repositório:** um script gera um relatório fictício com o mesmo layout e as mesmas posições do PDF real (cabeçalho, resumo e transações em duas linhas), com nomes e valores inventados. Esse arquivo e o resultado esperado são os testes golden e rodam no CI.
- **PDF real fora do git:** fica numa pasta ignorada pelo git, só na máquina do desenvolvedor, para conferir que o leitor bate com o relatório verdadeiro. O teste com o real é pulado quando o arquivo não existe.
- Nenhum dado pessoal (nome, telefone, e-mail) entra no repositório.

### 6. Primeiro passo
O leitor do PDF começa já, no WSL, sem esperar o Android Studio: o parser e os testes golden só precisam do Node. A tela de importação e o teste no emulador vêm depois da instalação do Android Studio (pendência 9 do `CLAUDE.md`).

### 7. Organização no GitHub
Os três repositórios (`Pilotei`, `Pilotei-Backend` e `Pilotei-Driver`) foram transferidos da conta `MarlosAraujo` para a organização **`Pilotei`** em 04/10/2026. Histórico, PRs e Actions foram junto, e o GitHub redireciona as URLs antigas. No plano gratuito da organização, os repositórios privados não têm proteção de branch, como antes.

## O que muda nas decisões anteriores
| Decisão | Antes | Agora |
|---|---|---|
| `0002` itens 1 e 3 | Kotlin nativo com Jetpack Compose; apps no Windows | React Native + Expo; código no WSL e emulador no Windows |
| `0003` (ambiente) | Apps Kotlin no Windows | Como acima |
| `0004` item 5 | Room como cache de leitura | Cache de leitura continua (só consulta sem internet); a biblioteca é escolhida na montagem do app. PROPOSTA: TanStack Query com persistência ou `expo-sqlite` |
| `0005` item 1 | Credential Manager do Android | O app obtém o token de ID do Google por uma biblioteca React Native. PROPOSTA: `@react-native-google-signin/google-signin`; PRECISA VALIDAR se ela usa o Credential Manager. O `core` não muda |
| `0005` item 12 | Billing Library | Biblioteca React Native da Google Play Billing. PROPOSTA: `expo-iap` ou `react-native-iap`; PRECISA VALIDAR o suporte a `obfuscatedAccountId` |
| Leitor do PDF | PdfBox-Android em Kotlin | Parser em TypeScript + pdf.js (seção 4) |

Continuam iguais: backend, autenticação no `core`, eventos, Google Play Billing como meio de assinatura no Android, internet obrigatória para lançar e todas as regras de produto.

## Motivos
- **Uma linguagem só:** TypeScript no app e no backend. Tipos e regras puras podem ser reaproveitados (por exemplo, os contratos HTTP).
- **iOS possível no pós-MVP** para a parte de gestão, sem reescrever o app.
- **O leitor do PDF em TypeScript** roda e é testado no Node, no mesmo ambiente do backend, sem esperar o Android Studio.

## Consequências
- **O Copiloto (MVP 2) continua nativo:** `AccessibilityService`, sobreposição (`SYSTEM_ALERT_WINDOW`), captura de tela e OCR são escritos em Kotlin, num Expo Module. Com o app em segundo plano, o JavaScript pode não estar rodando, então a leitura da oferta e o semáforo ficam no lado nativo. Isso era o principal motivo da 0002 e passa a ser custo aceito.
- O app de teste de acessibilidade (antes do MVP 2) vira um Expo Module de teste.
- A _development build_ exige o Android SDK ou o EAS Build. O Expo Go não serve para o Pilotei.
- Depender do pdf.js no Hermes é um risco técnico, com saída já definida (seção 4).
- O contrato entre app e backend continua explícito (OpenAPI gerado pelo NestJS). Compartilhar pacotes TypeScript entre os dois repositórios fica para quando houver necessidade.

## Em aberto
1. pdf.js no Hermes (seção 4).
2. _Development build_ local ou EAS Build (seção 3).
3. Bibliotecas de cache, login com Google e Billing (tabela acima), na montagem do app.
4. iOS no pós-MVP: assinatura pela App Store e conta Apple Developer.

## Referências
- Stack anterior: `docs/decisoes/0002-stack-apps-e-backend.md`
- Comparação: `docs/backend/comparacao-stack.md`
- Especificação do leitor: `docs/parser-uber-relatorio-semanal.md`
- Backend como fonte da verdade: `docs/decisoes/0001-backend-fonte-da-verdade.md`
