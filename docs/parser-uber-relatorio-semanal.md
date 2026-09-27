# Parser — Relatório semanal Uber (PDF)

Origem: drivers.uber.com › Ganhos › Relatórios semanais › baixar PDF. Validado com o relatório de 14/09 a 21/09/2026 (5 páginas, PDF gerado pelo Chromium/Skia).

## Estrutura do PDF
- Cabeçalho: "Relatório semanal" e o período "14 de set. de 2026 4h - 21 de set. de 2026 4h". A semana começa e termina às 04:00 de segunda.
- Dados pessoais (nome, telefone, e-mail): DESCARTAR, nunca salvar.
- Resumo semanal: saldo inicial, "Seus ganhos" (total) e saldo final.
- Detalhamento de Seus ganhos: Valor (Base, Preço dinâmico, Tempo na parada, UberX Prioridade, Tempo de espera na partida) e Promoção (Ganhos Uber Pro, Promoção). Só existe como total da semana, não por corrida.
- Transações (a parte que mapeamos). Cada transação ocupa duas linhas de texto:
  - Linha 1: Processado (dia da semana e data) | Evento (tipo) | Seus ganhos | Saldo (do evento)
  - Linha 2: Processado (hora) | Data e hora do evento | Saldo acumulado

## Mapeamento
| Campo do PDF | Destino | Observação |
|---|---|---|
| Evento = "Uber X" | TripImport.category = "UberX" | |
| Evento = "Prioridade" | TripImport.category = "UberX Prioridade" | |
| Evento = "Promoção - ..." | Income(type=BONUS), não TripImport | Ex.: R$ 75,00 extra por concluir 5 viagens |
| Data e hora do evento | TripImport.started_at | Provável início ou aceite. PRECISA VALIDAR |
| Processado (data e hora) | TripImport.finished_at (aprox.) | Diferença típica de 10 a 36 min |
| Seus ganhos | TripImport.net_amount | Já líquido, com dinâmico e Uber Pro rateados |
| Saldo acumulado | só para conferência | Valida a ordem e a leitura |
| Linha sem "Seus ganhos" (R$ 0,00) | Ajuste; vincular ao evento de mesma data e hora | Ex.: evento 18/09 18:57, processado em 20/09 |

- Não vem: distância, duração exata, origem, destino, ID da viagem, taxa da Uber, valor pago pelo passageiro.
- O ano não aparece nas linhas: tirar do período do cabeçalho, com cuidado na virada de ano.
- Chave de deduplicação: hash de (UBER, data e hora do evento, categoria, valor). Não existe ID da viagem.

## Validações obrigatórias antes de importar
1. A soma de "Seus ganhos" das transações deve ser igual ao total "Seus ganhos" do resumo. No exemplo: 33 linhas com valor = R$ 420,18 ✓.
2. Saldo inicial + ganhos = saldo final (-0,01 + 420,18 = 420,17 ✓).
3. O saldo acumulado de cada linha deve ser coerente com o da anterior. Se não for, avisar que a leitura falhou.
4. Se alguma validação falhar, mostrar a prévia com aviso e não importar em silêncio.

## Números do exemplo
- 32 corridas (UberX e Prioridade), 1 promoção de R$ 75,00 e 1 ajuste de R$ 0,00.
- O detalhamento separa R$ 28,95 de Uber Pro, mas esse valor não aparece como linha própria nas transações: está embutido nos valores das corridas.

## Implementação Android
- Extrair o texto com posição com PdfBox-Android (com.tom_roush:pdfbox-android). O PdfRenderer nativo só gera imagem, o que exigiria OCR.
- Separar colunas pela coordenada X, sem depender de espaços.
- Parser por versão de layout: detectar pelos títulos ("Relatório semanal", "Transações", "Processado", "Evento"). Se o layout for desconhecido, falhar de forma explícita e oferecer a digitação manual.
- Guardar PDFs de exemplo anonimizados como fixtures de teste (golden tests).

## Limitações
- É semanal: não serve para o fechamento do dia. PRECISA VALIDAR se o portal gera o relatório da semana em andamento.
- Como não traz KM, o KM real continua vindo do lançamento "KM do dia" (odômetro) no Pilotei.
- A duração estimada (processado − evento) serve só para a análise por horário, marcada como estimada.
