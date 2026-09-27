# Backend — glossário de entidades

- **Atualizado em:** 27/09/2026

Cada entidade do backend responde a três perguntas: **o que é**, **para que serve** e **como utilizar**. Os domínios estão em `dominios.md`. Campos citados aqui vêm do planejamento (`CLAUDE.md`); os demais serão definidos na modelagem.

## 1. Identidade

### `User`
- **O que é:** o motorista cadastrado no Pilotei.
- **Para que serve:** é o dono de todos os dados (veículo, custos, ganhos, metas, assinatura). Sem ele, nada é gravado.
- **Como utilizar:** criado no primeiro login (Google ou e-mail + OTP). O mesmo e-mail verificado leva ao mesmo `User`, seja pelo Google, seja pelo OTP. Todo registro do motorista referencia o `User`. A exclusão de conta apaga o `User` e todos os seus dados; a exportação gera uma cópia deles (LGPD).

### `Session`
- **O que é:** uma sessão de login ativa em um aparelho.
- **Para que serve:** prova que as chamadas do app vêm do motorista autenticado, sem pedir novo código ou novo login com Google a cada uso.
- **Como utilizar:** criada no login e encerrada ao sair, ao expirar ou ao excluir a conta. Guarda o hash do refresh token (30 dias, trocado a cada uso). Cada requisição do app leva o access token (JWT de 15 minutos). Ver `docs/decisoes/0005-autenticacao.md`.

### `EmailOtp`
- **O que é:** um código de uso único enviado por e-mail (pelo Resend) para o motorista entrar sem senha.
- **Para que serve:** comprova que o motorista é dono do e-mail. O Pilotei não usa senha, então não há recuperação de senha.
- **Como utilizar:** criado quando o motorista pede o código; guarda só o hash, a validade e as tentativas. Usado uma vez e descartado. Parâmetros: 6 dígitos, 10 minutos, 5 tentativas, novo envio após 60 segundos.

## 2. Assinaturas

### `Subscription`
- **O que é:** a assinatura do motorista na Google Play (plano mensal ou anual) e seu estado: teste, ativa, cancelada (não renova) ou expirada.
- **Para que serve:** guarda a situação real da assinatura, confirmada pelo servidor junto à Play, e não pelo que o app diz.
- **Como utilizar:** criada quando o app envia a compra para validação; atualizada pelas notificações da Play (renovação, cancelamento, expiração). O teste grátis de 7 dias só vale para quem nunca assinou.

### `Entitlement`
- **O que é:** o direito de uso que a assinatura concede, por exemplo "pode fazer lançamentos".
- **Para que serve:** separa "o que o motorista comprou" (`Subscription`) de "o que ele pode fazer agora". O resto do backend consulta só o `Entitlement`.
- **Como utilizar:** recalculado sempre que a `Subscription` muda. Toda gravação de dados verifica se há um `Entitlement` ativo; consulta e exportação não exigem.

## 3. Veículos

### `Vehicle`
- **O que é:** o carro ou a moto que o motorista usa para trabalhar (marca, modelo, ano, combustível, consumo médio, uso pessoal sim/não).
- **Para que serve:** é a base do custo por km: consumo, manutenção e custos fixos são do veículo.
- **Como utilizar:** cadastrado na configuração inicial (tela Veículo). O consumo informado é a estimativa inicial, corrigida depois pelos abastecimentos de tanque cheio.

### `OdometerReading`
- **O que é:** uma leitura do KM do painel em um momento (por exemplo, KM inicial e final do dia).
- **Para que serve:** dá o KM real rodado, que o PDF da Uber não traz, e separa o KM de trabalho do uso pessoal.
- **Como utilizar:** registrado no lançamento "KM do dia" ou no abastecimento. A diferença entre leituras dá os km rodados.

## 4. Custos

### `FuelEntry`
- **O que é:** um abastecimento (litros, preço por litro, valor total, KM do painel, tanque cheio sim/não).
- **Para que serve:** compõe o custo de combustível e, com dois tanques cheios seguidos, calcula o consumo real.
- **Como utilizar:** lançado na tela "Novo lançamento › Combustível". O motorista preenche 2 dos 3 valores e o terceiro é calculado. Conta como saída de caixa.

### `Maintenance`
- **O que é:** uma manutenção feita ou prevista (troca de óleo, pneus, revisão), com valor e KM.
- **Para que serve:** compõe o custo de manutenção e pneus por km e gera os avisos de manutenção preventiva ("Troca de óleo em 9.344 km").
- **Como utilizar:** lançada em "Novo lançamento › Manutenção". As manutenções previstas usam o KM atual do veículo para avisar.

### `FixedCost`
- **O que é:** um custo fixo do veículo: seguro, IPVA e licenciamento, financiamento, aluguel ou outros mensais.
- **Para que serve:** entra no custo por km, rateado pelos km rodados no mês.
- **Como utilizar:** cadastrado na tela Custos fixos, com valor e periodicidade (mensal ou anual).

### `Expense`
- **O que é:** uma despesa avulsa que não é combustível nem manutenção (lavagem, estacionamento, pedágio).
- **Para que serve:** entra no resultado de caixa do dia.
- **Como utilizar:** lançada em "Novo lançamento › Despesa" ou em "Custos do dia › Outra despesa".

## 5. Ganhos

### `TripImport`
- **O que é:** uma corrida feita em outra plataforma (Uber, 99, inDrive, particular), concluída ou cancelada com taxa, trazida por importação ou lançada à mão: plataforma, categoria, início, fim, valor líquido, gorjeta, origem do dado (arquivo ou manual).
- **Para que serve:** é a menor unidade de ganho; permite análises por horário e categoria.
- **Como utilizar:** criada pela importação do PDF da Uber ou lançada à mão. Duplicatas são barradas pelo ID externo ou, sem ele, pelo hash (plataforma, data e hora do evento, categoria, valor).

### `Income`
- **O que é:** um ganho que não é corrida, como promoções e bônus (tipo BONUS).
- **Para que serve:** soma ao faturamento sem ser contado como corrida, para não distorcer R$/km e corridas por hora.
- **Como utilizar:** criada pela importação (linhas "Promoção - ...") ou à mão.

### `PlatformDaySummary`
- **O que é:** o resumo de um dia lançado à mão em uma plataforma: data, ganhos, KM, horas online e número de corridas.
- **Para que serve:** registra o dia sem inventar corridas individuais.
- **Como utilizar:** criado na tela "Adicionar dia". Quando um PDF da Uber cobre o mesmo período, ele substitui os resumos manuais da Uber, para não contar em dobro.

### `PlatformConnection`
- **O que é:** como o motorista traz os ganhos de cada plataforma (Uber por PDF, 99 manual, inDrive manual, particulares manual) e o estado disso.
- **Para que serve:** monta a tela "Importar ganhos" e guarda a data da última importação.
- **Como utilizar:** criada por plataforma na configuração. Métodos novos são liberados por feature flag.

## 6. Importação

### `ImportBatch`
- **O que é:** um lote de importação: origem (por exemplo, relatório semanal da Uber), hash do arquivo, período, contagens e erros.
- **Para que serve:** agrupa tudo que entrou de uma vez, evita importar o mesmo arquivo duas vezes e permite desfazer a importação.
- **Como utilizar:** o app lê o PDF no celular e envia ao backend as corridas, promoções e ajustes, junto com o hash e as contagens. O backend cria o lote e liga cada `TripImport` e `Income` a ele. Desfazer remove o lote inteiro.

## 7. Métricas

### `CostSnapshot`
- **O que é:** o custo por km calculado em um momento, com a composição (combustível, manutenção e pneus, custos fixos), o nível de confiança (baixo, médio ou alto) e a base usada (estimado, observado ou personalizado).
- **Para que serve:** o dashboard e o resultado econômico usam o custo vigente, e o motorista vê de onde ele veio.
- **Como utilizar:** recalculado pelo servidor quando entram abastecimentos, manutenções, custos fixos ou KM. Os resultados de um dia usam o custo vigente naquele dia.

### `Goal`
- **O que é:** uma meta do motorista: R$/km mínimo, R$/hora mínimo, resultado diário, horas por dia, KM por mês.
- **Para que serve:** compara o desempenho no dashboard e, no MVP 2, alimenta o semáforo do Copiloto.
- **Como utilizar:** cadastrada na tela Metas. O KM por mês é usado no rateio dos custos fixos enquanto não há histórico.

## 8. Relatórios
Não tem entidade própria. Os relatórios de Hoje, Semana e Mês e o dashboard são calculados a partir de `TripImport`, `Income`, `PlatformDaySummary`, custos e `CostSnapshot`.

## 9. Config remota e flags

### `FeatureFlag`
- **O que é:** uma chave liga/desliga de funcionalidade (por exemplo, liberar a importação da 99).
- **Para que serve:** ativa ou desativa recursos sem publicar nova versão do app.
- **Como utilizar:** o app consulta as flags ao abrir e mostra ou esconde o recurso.

### `RemoteConfig`
- **O que é:** um valor de configuração mantido no servidor (por exemplo, a URL do portal da Uber ou um aviso aos usuários).
- **Para que serve:** muda textos e endereços sem nova versão do app.
- **Como utilizar:** lido pelo app junto com as flags.

## 10. Notificações

### `Notification`
- **O que é:** um aviso ao motorista, como manutenção próxima ou fim do teste grátis (2 dias antes).
- **Para que serve:** lembra o motorista do que exige ação.
- **Como utilizar:** gerada pelo backend a partir de regras (KM do veículo, datas da assinatura) e entregue por push ou dentro do app.

## 11. Copiloto (MVP 2)

### `Offer`
- **O que é:** uma oferta de corrida lida na tela do app da plataforma (valor, distância, tempo) e a análise feita (semáforo).
- **Para que serve:** compara depois o que foi oferecido com o resultado real, para melhorar a análise.
- **Como utilizar:** a leitura e o semáforo acontecem no celular, em tempo real. O app envia as ofertas ao backend depois. O valor da oferta da Uber já vem sem a taxa da plataforma.

## 12. Corridas do Pilotei APP (pós-MVP)

### `Trip`
- **O que é:** uma corrida (viagem) pedida por um passageiro no Pilotei APP e feita por um motorista do Pilotei Driver.
- **Para que serve:** registra o pedido, o motorista, o trajeto, o preço, o pagamento do passageiro e a taxa de R$ 1,00 do Pilotei.
- **Como utilizar:** criada quando o passageiro pede a corrida, no serviço `trip`. Campos, estados e regras serão definidos na modelagem do pós-MVP. Não confundir com `TripImport`, que guarda corridas de outras plataformas.
