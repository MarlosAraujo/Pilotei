# 0005 — Autenticação

- **Data:** 27/09/2026
- **Status:** aprovada (pontos em aberto listados abaixo)
- **Branch:** `003-setup-infra`

## Contexto
A autenticação define como o `userId` circula do `gateway` para os serviços e, com isso, o formato de todas as mensagens Kafka. As opções comparadas foram: implementação própria no `core`, Firebase Authentication e Keycloak em container. A decisão `0004-ambiente-e-containers.md` pede não amarrar o Pilotei a um provedor.

## Decisão

### Implementação própria, no serviço `core`
1. **Login com Google:** o app obtém o token de ID do Google pelo Credential Manager do Android (CONFIRMADO) e o envia ao `core`, que o valida com `google-auth-library` (CONFIRMADO).
2. **Login com e-mail e OTP, sem senha:** o motorista informa o e-mail e recebe um código de uso único, enviado pelo **Resend**. Não há senha, e por isso não há recuperação de senha: esqueceu, pede outro código.
3. **E-mail verificado antes do primeiro lançamento:** atendido pelos dois métodos. O login por OTP comprova o e-mail; no Google, o token traz o e-mail verificado.

### Tokens e sessões
4. **Access token:** JWT de **15 minutos**, assinado com chave assimétrica pelo `core`, com `sub = userId` (UUID).
5. **Refresh token:** opaco, de **30 dias**, trocado a cada uso e guardado só como hash na `Session` (uma por aparelho).
6. **Validação:** o `gateway` valida o JWT localmente, com a chave pública do `core`, sem chamar o `core` a cada requisição.
7. **Sair ou excluir conta:** apaga a `Session`. O access token já emitido deixa de valer em até 15 minutos.

### `userId` nos eventos
8. O `gateway` repassa o `userId` aos serviços dentro do envelope da mensagem Kafka:
   `{ eventId, type, version, occurredAt, userId, correlationId, payload }`
9. O `userId` é a **chave de partição** no Kafka, para manter a ordem dos eventos de cada motorista.
10. Os serviços internos confiam no `userId` do envelope porque só o `gateway` fica exposto aos apps.
11. **Exclusão de conta:** o `core` publica `user.deleted`, e cada serviço apaga os dados do motorista no próprio schema (LGPD).

### Google Play Billing
12. A compra leva um hash do `userId` em `obfuscatedAccountId` (campo da Billing Library, CONFIRMADO), para o `core` ligar a assinatura ao usuário.

## Parâmetros (aprovados em 27/09/2026)
- **OTP:** 6 dígitos, válido por 10 minutos, guardado só como hash, no máximo 5 tentativas por código, novo envio após 60 segundos e limite de envios por e-mail e por IP por hora.
- **Conta única por e-mail:** Google e OTP com o mesmo e-mail verificado entram no mesmo `User`.
- **Entidade `EmailOtp`** no domínio Identidade (e-mail, hash do código, validade, tentativas).

## Consequências
- O Pilotei não guarda senhas: sem vazamento de senha e sem fluxo de troca de senha.
- O login por e-mail depende da entrega do Resend. Atraso ou e-mail na caixa de spam atrasa o acesso; o login com Google fica como alternativa.
- O Resend exige **domínio próprio verificado** por registros DNS para enviar e-mails. Domínio escolhido: **`pilotei.app.br`** (verificação no Resend a fazer). PRECISA VALIDAR plano e limites do Resend.
- As telas Criar conta e Entrar do protótipo (`design/`) foram ajustadas: sem campo de senha, com o botão "Enviar código".
- O `core` guarda a chave privada de assinatura dos JWT; a chave pública é publicada para o `gateway`. A troca periódica de chaves fica para a implementação.

## Em aberto
1. **Passageiros do Pilotei APP:** mesma tabela `User` com papel ou cadastro separado. Decidir no pós-MVP.

## Referências
- Ambiente e containers: `docs/decisoes/0004-ambiente-e-containers.md`
- Domínios: `docs/backend/dominios.md`
- Glossário: `docs/backend/glossario-entidades.md`
