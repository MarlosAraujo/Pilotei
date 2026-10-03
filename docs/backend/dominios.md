# Backend — divisão em domínios

- **Atualizado em:** 27/09/2026
- **Status:** aprovada (decisão `docs/decisoes/0001-backend-fonte-da-verdade.md`)

O backend é a fonte da verdade e se divide em 12 domínios (o 12º, pós-MVP, atende o Pilotei APP). O significado de cada entidade (o que é, para que serve, como utilizar) está em `glossario-entidades.md`.

| # | Domínio | Responsável por | Entidades | Fase |
|---|---|---|---|---|
| 1 | **Identidade** | Cadastro, login com Google e com e-mail + OTP (sem senha, envio pelo Resend), sessões e tokens, exclusão de conta e exportação de dados (LGPD). Ver `docs/decisoes/0005-autenticacao.md` | `User`, `Session`, `EmailOtp` | MVP 1 |
| 2 | **Assinaturas** | Validar as compras da Google Play, receber as notificações da Play e decidir quem pode fazer lançamentos | `Subscription`, `Entitlement` | MVP 1 |
| 3 | **Veículos** | Veículo, odômetro e uso pessoal | `Vehicle`, `OdometerReading` | MVP 1 |
| 4 | **Custos** | Abastecimentos, manutenção (inclusive preventiva), custos fixos e despesas | `FuelEntry`, `Maintenance`, `FixedCost`, `Expense` | MVP 1 |
| 5 | **Ganhos** | Plataformas, corridas, bônus, dias lançados à mão e conexões com plataformas | `TripImport`, `Income`, `PlatformDaySummary`, `PlatformConnection` | MVP 1 |
| 6 | **Importação** | Receber as corridas já lidas no celular, deduplicar e permitir desfazer uma importação | `ImportBatch` | MVP 1 |
| 7 | **Métricas** | Custo/km com nível de confiança, resultado econômico e de caixa, ganho real por hora e metas | `CostSnapshot`, `Goal` | MVP 1 |
| 8 | **Relatórios** | Visões de Hoje, Semana e Mês e o dashboard | Nenhuma própria: agregações das entidades acima | MVP 1 |
| 9 | **Config remota e flags** | URLs dos portais, feature flags e avisos | `FeatureFlag`, `RemoteConfig` | MVP 1 |
| 10 | **Notificações** | Lembretes de manutenção e aviso de fim do teste grátis | `Notification` | MVP 1 ou depois |
| 11 | **Copiloto** | A leitura de ofertas fica no celular; o servidor recebe as ofertas depois, para comparar oferta com resultado | `Offer` | MVP 2 |
| 12 | **Corridas do Pilotei APP** | Pedido do passageiro, despacho ao motorista do Pilotei Driver, preço, pagamento pelo app e taxa de R$ 1,00 | `Trip` | Pós-MVP |

## Serviços (decisão `docs/decisoes/0003-microservices-kafka.md`)
| Serviço | Porta | Domínios |
|---|---|---|
| `gateway` | 7000 | BFF (HTTP para os apps, autenticação) |
| `core` | 7100 | 1 Identidade, 2 Assinaturas |
| `vehicle` | 7200 | 3 Veículos, 4 Custos, 5 Ganhos, 6 Importação |
| `reports` | 7300 | 7 Métricas, 8 Relatórios |
| `notification` | 7400 | 9 Config remota e flags, 10 Notificações |
| `copilot` | 7500 | 11 Copiloto (MVP 2) |
| `trip` | 7600 | 12 Corridas do Pilotei APP (pós-MVP): entidade `Trip` |

Comunicação entre serviços por Kafka; um schema por serviço num PostgreSQL único. Todos rodam em Docker. Todos os serviços se conectam ao mesmo Kafka, cada um com seus tópicos. As demais entidades do domínio 12 ainda PRECISAM SER DEFINIDAS.

**Nomes:** `Trip` = corrida/viagem do Pilotei APP (serviço `trip`). `TripImport` = corrida de outra plataforma, importada ou lançada à mão (serviço `vehicle`, domínio Ganhos).

## Regras gerais
- Dinheiro sempre em centavos (inteiro). Datas em UTC, exibidas no fuso America/Sao_Paulo.
- Todo dado do motorista pertence a um `User`, e só ele o acessa.
- Lançar dados exige internet e um `Entitlement` ativo. Consulta e exportação continuam liberadas com a assinatura expirada.
- O Dashboard soma `TripImport`, `Income` e `PlatformDaySummary` sem contar nada em dobro. O PDF semanal da Uber substitui os dias lançados à mão da Uber no mesmo período.

## Fora da divisão
- **Sincronização:** descartada. Com internet obrigatória para lançar, não há fila offline nem conflitos. Volta como nova decisão se o uso offline for retomado.
- **Leitura do PDF:** acontece no app, não no backend.
