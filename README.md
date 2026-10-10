# Pilotei

App Android para motoristas de aplicativo (Uber, 99, inDrive e corridas particulares) que mostra quanto realmente sobra do trabalho: junta gestão do veículo, custos reais, ganhos por plataforma e análise de ofertas.

> Status: fase de definição. Este repositório guarda o planejamento; o código vai para o monorepo [`Pilotei/Pilotei-Driver`](https://github.com/Pilotei/Pilotei-Driver).

## Stack (revisão de 10/10/2026)

- **Monorepo único** `Pilotei/Pilotei-Driver`, com o app e o backend.
- **App:** React Native + Expo (development build), só Android no MVP. Código nativo do Copiloto em Kotlin.
- **Backend:** NestJS em monólito modular, Prisma 7 e PostgreSQL, em VPS Hetzner CPX22.
- **Login:** Google ou e-mail + código (OTP), sem senha.
- **Assinatura:** Google Play Billing; R$ 10,99/mês ou R$ 97,99/ano, com 7 dias grátis.

## Estrutura do repositório

- `CLAUDE.md`: contexto completo do projeto (produto, decisões, roadmap, modelo de dados, pendências).
- `docs/prompt-mestre.md`: prompt mestre de desenvolvimento para o Claude Code, base do monorepo.
- `docs/decisoes/`: decisões registradas até aqui (algumas substituídas pela revisão do prompt mestre).
- `docs/parser-uber-relatorio-semanal.md`: especificação do leitor do relatório semanal da Uber em PDF.
- `design/`: protótipo das 18 telas (HTML navegável e fonte do canvas).

## Fluxo de trabalho

- Cada feature tem uma branch numerada criada a partir de `develop` (`001-setup/estrutura-inicial`, `002-setup-stack`, ...).
- Ao final, um PR para `develop` descreve o que foi implementado e o que deve ser testado.

O Pilotei não é afiliado à Uber, 99 ou inDrive.
