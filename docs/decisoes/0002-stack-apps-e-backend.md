# 0002 — Stack dos apps e do backend

- **Data:** 27/09/2026
- **Status:** aprovada; itens 1 e 3 substituídos pela `0008-react-native-e-leitor-pdf.md` (04/10/2026)
- **Branch:** `002-setup-stack`

## Contexto
Com o backend definido como fonte da verdade (`0001-backend-fonte-da-verdade.md`), faltava escolher as linguagens. Para os apps, as opções foram React Native + Expo e Kotlin nativo. O Copiloto (MVP 2) exige serviço de acessibilidade, sobreposição de tela e leitura rápida da oferta, e o leitor do PDF da Uber usa uma biblioteca Java (PdfBox-Android). A comparação completa está em `docs/backend/comparacao-stack.md`.

## Decisão
1. **Apps em Kotlin nativo**, com Jetpack Compose (substituído: apps em React Native + Expo, ver `0008`):
   - **Pilotei Driver:** app do motorista. Gestão (veículo, custos, ganhos, importação, dashboard) e Copiloto (acessibilidade, sobreposição e semáforo).
   - **Pilotei APP:** app do usuário final (passageiro), que chama um motorista do Pilotei Driver para uma corrida. **Fase: pós-MVP**, no serviço `trip`.
2. **Backend em NestJS (Node + TypeScript), em microservices.** Divisão em `0003-microservices-kafka.md`.
3. **Ambiente de desenvolvimento** (substituído para os apps, ver `0008`): apps Kotlin no Windows (Android Studio + emulador); backend no WSL2 (Ubuntu 24).
4. **Receita:**
   - **Pilotei Driver:** assinatura do motorista (Google Play Billing, como já definido).
   - **Pilotei APP:** **R$ 1,00 por corrida realizada**, pago pelo **passageiro, pelo próprio app**. Por ser serviço físico, a corrida não usa Google Play Billing; exige um meio de pagamento (gateway) a escolher.

## Motivos
- Acessibilidade (`AccessibilityService`), sobreposição (`SYSTEM_ALERT_WINDOW`), captura de tela e OCR são nativos do Android. No React Native, essa parte seria escrita em Kotlin do mesmo jeito, com uma ponte JS↔nativo a mais para manter.
- O leitor do PDF (PdfBox-Android) é uma biblioteca Java e roda direto no Kotlin.
- O iOS não permite ler a tela de outro app nem desenhar por cima dele, então o ganho multiplataforma do React Native não alcança o Copiloto.
- NestJS: organização em módulos que casa com os domínios (`docs/backend/dominios.md`), ecossistema grande e suporte nativo a microservices (`@nestjs/microservices`).

## Consequências
- **Duas linguagens:** Kotlin nos apps e TypeScript no backend. O contrato entre eles precisa ser explícito (por exemplo, OpenAPI gerado pelo NestJS), já que não há modelos compartilhados.
- **Microservices trazem custo operacional:** vários deploys, comunicação entre serviços, observabilidade e dados distribuídos. A granularidade inicial precisa ser definida com cuidado para o MVP 1.
- **Ambiente:** com o código dos apps no disco do Windows, o Android Studio evita a lentidão de abrir projetos dentro do WSL. O backend fica no WSL.
- **Pilotei APP muda o escopo do produto:** o Pilotei deixa de ser só uma ferramenta para motoristas e passa a intermediar corridas. Isso traz domínios novos (pedido de corrida, despacho, localização em tempo real, preço, pagamento do passageiro, avaliação) e pontos regulatórios e de loja. Ver "Em aberto".
- **Substitui** a recomendação de `comparacao-stack.md` (Node + TS + Prisma com um serviço único).

## Em aberto
1. **Pilotei APP:** gateway de pagamento, repasse ao motorista e campos da entidade `Trip`.
2. **Regulação e loja:** transporte individual por app (Lei 13.640/2018 e leis municipais), cadastro e verificação de motoristas, políticas da Google Play para apps de transporte. PRECISA VALIDAR.
3. **Autenticação e hospedagem.** O ORM foi definido em `0003-microservices-kafka.md` (Prisma v7).

## Revisão de 04/10/2026
- **Apps trocados para React Native + Expo** (`0008-react-native-e-leitor-pdf.md`): só Android no MVP e iOS no pós-MVP, sem o Copiloto. O código nativo do Copiloto continua em Kotlin, num Expo Module. Os itens 2 e 4 (backend e receita) não mudam.

## Referências
- Comparação: `docs/backend/comparacao-stack.md`
- Domínios: `docs/backend/dominios.md`
