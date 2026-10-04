# Comparação de stack — apps e backend

- **Atualizado em:** 27/09/2026
- **Status:** decidido em `docs/decisoes/0002-stack-apps-e-backend.md` (backend em NestJS com microservices). Os apps eram Kotlin e passaram para React Native + Expo em `docs/decisoes/0008-react-native-e-leitor-pdf.md` (04/10/2026).
- Preços de memória (2025): PRECISAM SER VALIDADOS.

## 1. Apps: React Native + Expo × Kotlin nativo

### Necessidades do Pilotei
| Necessidade | React Native + Expo | Kotlin nativo |
|---|---|---|
| **Acessibilidade** (ler a oferta na tela) | `AccessibilityService` é componente nativo, declarado no manifesto com XML de configuração. Exige **módulo nativo em Kotlin** (Expo Modules API + config plugin) e *development build*; o Expo Go não roda código nativo próprio. CONFIRMADO | Suporte direto, inclusive `takeScreenshot()` (Android 11+) e OCR com ML Kit. CONFIRMADO |
| **Sobreposição** (semáforo sobre o app da Uber) | `SYSTEM_ALERT_WINDOW` + `WindowManager` a partir de um Service nativo. Renderizar React Native numa janela flutuante é trabalhoso; na prática, a sobreposição seria escrita em Kotlin. Bibliotecas da comunidade: manutenção PRECISA VALIDAR | Suporte direto (View ou Compose; o Compose pede configurar o ciclo de vida, padrão conhecido). CONFIRMADO |
| **Notificações nativas** | `expo-notifications` (locais e push via FCM). CONFIRMADO | `NotificationCompat` + FCM. CONFIRMADO |

As duas pedem a permissão `POST_NOTIFICATIONS` em tempo de execução no Android 13+.

### Outros pontos
- **Leitor do PDF da Uber:** a especificação usa PdfBox-Android (Java, posições X do texto). No React Native seria mais um módulo nativo.
- **Latência do Copiloto:** com o app em segundo plano, o JavaScript pode não estar rodando. A leitura e o semáforo teriam de ser nativos também no React Native.
- **iOS:** o iOS não permite ler a tela de outro app nem desenhar sobre ele (CONFIRMADO). Só o MVP 1 seria portável.
- **Billing e voz:** existem nas duas (`react-native-iap`/`expo-iap` e `expo-speech`; Billing Library e `TextToSpeech`).

### Ambiente (Windows + emulador, código no WSL2)
- **Kotlin:** Android Studio no Windows abrindo projeto em `\\wsl$` costuma ser lento no Gradle e na indexação. Saídas: código dos apps no disco do Windows, ou Gradle no WSL com o `adb` ligado ao emulador do Windows. PRECISA VALIDAR.
- **React Native + Expo:** Metro roda bem no WSL, mas também precisa do `adb` ligado ao emulador (`adb reverse`). *Development build* local exige Android SDK no WSL, ou EAS Build na nuvem (cota no plano grátis).

### Resultado
**Kotlin** em 27/09/2026, trocado por **React Native + Expo** em 04/10/2026 (`0008`): uma linguagem só com o backend e iOS possível no pós-MVP. O Copiloto continua nativo, num Expo Module. Argumento original: o diferencial do Pilotei (acessibilidade, sobreposição, OCR, leitor de PDF) é nativo; no React Native ele seria escrito em Kotlin mesmo, com a ponte a mais.

## 2. Backend

| Critério | Node + TypeScript | Kotlin + Ktor ou Spring | Supabase |
|---|---|---|---|
| **Validar assinaturas da Google Play** | Biblioteca oficial `googleapis` (Android Publisher API); notificações da Play por Pub/Sub num endpoint próprio | Biblioteca oficial Java `google-api-services-androidpublisher`, usada em Kotlin; mesmo fluxo | Nada pronto: Edge Functions (TypeScript/Deno) chamando a mesma API e recebendo o Pub/Sub |
| **Login com Google** | `google-auth-library` valida o token do app; e-mail/senha, recuperação e sessões por conta própria ou biblioteca | `GoogleIdTokenVerifier` oficial; o resto também por conta própria | Pronto: Google, e-mail/senha, recuperação e exclusão de usuário; Android usa `signInWithIdToken` |
| **Custo de hospedagem** | Baixo: ~US$ 5–20/mês (Railway, Render, Fly.io) + Postgres | Médio: JVM pede 512 MB–1 GB, ~US$ 10–30/mês; Spring pesa mais que Ktor | Grátis para desenvolver; Pro ~US$ 25/mês por projeto |
| **Facilidade de manutenção** | Maior ecossistema, fácil contratar; linguagem diferente do app | Mesma linguagem do app; Ktor leve, Spring completo e verboso | Menos código no início; regras de negócio espalhadas entre SQL, RLS e Edge Functions |
| **Aderência aos 11 domínios** | Boa: um módulo por domínio (NestJS ou Fastify) | Boa, mesma organização | Parcial: bom para Identidade e dados, fraco para regras de negócio |

### Resultado
**NestJS (Node + TypeScript) em microservices**, escolhido pelo responsável pelo produto. A recomendação anterior era um serviço único (Node + TS + Prisma + Postgres). Com microservices, os custos de hospedagem acima passam a valer por serviço.
