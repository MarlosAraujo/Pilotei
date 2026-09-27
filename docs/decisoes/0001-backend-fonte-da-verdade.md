# 0001 — Backend como fonte da verdade

- **Data:** 26/09/2026
- **Status:** aprovada
- **Branch:** `002-setup-stack`

## Contexto
A arquitetura proposta no planejamento era offline-first: dados no celular (Room) e backup opcional. O responsável pelo produto decidiu que o Pilotei terá um backend e que ele será a fonte da verdade. Antes de qualquer código, toda a divisão do backend deve ser validada.

## Decisão
1. **O backend é a fonte da verdade.** O app Android é um cliente: o que ele guarda localmente é só cache de leitura.
2. **Responsabilidades do backend:** conta e login, validação de assinaturas, backup e sincronização dos dados do motorista, configuração remota e feature flags.
3. **Internet obrigatória para lançamentos (decisão A).** Todo lançamento (abastecimento, KM, despesa, ganhos, importação) exige conexão. Sem sinal, o app mostra os últimos dados carregados, só para consulta.
4. **O PDF da Uber é lido no celular (decisão B).** O arquivo nunca sai do aparelho. O app lê o PDF, mostra a prévia, e envia ao backend só as corridas, promoções e ajustes já lidos, junto com o hash do arquivo e as contagens do lote.
5. **As métricas são calculadas no servidor (decisão C).** Custo/km, nível de confiança, resultado econômico, resultado de caixa e ganho real por hora vêm do backend. O app não repete essa lógica.

## Consequências
- **Mais simples:** não há fila offline, resolução de conflitos nem lógica de métricas duplicada. O domínio de Sincronização foi descartado.
- **Sem sinal, não há lançamento.** Não é considerado risco: os apps das plataformas (Uber, 99, inDrive) também não funcionam sem internet, então o motorista já trabalha conectado.
- **Privacidade:** como o PDF não chega ao servidor, os dados pessoais do relatório (nome, telefone, e-mail) nunca são transmitidos.
- **Leitor do PDF no app:** um ajuste no layout do relatório da Uber exige nova versão do app. Mitigação: detectar layout desconhecido, falhar de forma explícita e oferecer a digitação manual (ver `docs/parser-uber-relatorio-semanal.md`).
- **Substitui** a linha "offline-first. Os dados ficam no celular e o backup é opcional" da arquitetura proposta no `CLAUDE.md`.

## Referências
- Divisão em domínios: `docs/backend/dominios.md`
- Glossário de entidades: `docs/backend/glossario-entidades.md`
