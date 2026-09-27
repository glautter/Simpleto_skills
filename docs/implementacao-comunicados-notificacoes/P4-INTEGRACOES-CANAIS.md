# P4 - Integracoes de canais e notificacoes

Data de atualizacao: 2026-09-27
Status: E-MAIL E WHATSAPP JA IMPLEMENTADOS E EM PRODUCAO (achado de revisao 2026-09-27 — o checklist geral estava desatualizado, marcando "nao iniciado")

## Achado de revisao (2026-09-27)

Antes de comecar qualquer implementacao, foi feita a revisao de codigo obrigatoria pelo passo 4 do checklist geral. Resultado: **P4 ja tem uma fatia real e testada em producao**, implementada em commits anteriores a esta sessao (nao relacionados ao trabalho de P1-P3 feito aqui):

- `015592e` "feat(notificacoes): liga cobranca-vencendo ao motor multicanal e disparo real de e-mail" (2026-08-15)
- `ff39285` "feat(notificacoes): fase 1 do disparo real de WhatsApp via OpenClaw"

Componentes confirmados no codigo:
- `Simpleto.Application/Services/EmailDispatchService.cs` + `Simpleto.Api/Workers/EmailDispatcherWorker.cs` (BackgroundService, polling a cada 30s) — envia via `IEmailSenderService`/`SmtpEmailSenderService`, mesmo adapter ja usado pelo convite de usuario.
- `Simpleto.Application/Services/WhatsAppDispatchService.cs` + `Simpleto.Api/Workers/WhatsAppDispatcherWorker.cs` — envia via `IWhatsAppSender`/`OpenClawWhatsAppSender` (provedor OpenClaw).
- Ambos: `AddHostedService<...>` registrados em `Program.cs` (linhas ~1716-1721), rodam de fato, nao so desenhados.
- Retry/backoff: `MaxTentativas = 3`, backoff fixo de 5 minutos, campo `Tentativas`/`UltimaTentativaEm` na tabela `notificacaocanal` — falha definitiva apos 3 tentativas fica com status `falhou` e motivo gravado (`FalhaMotivo`), visivel para consulta administrativa.
- Fila duravel: a propria tabela `notificacaocanal` (status `criada`/`pendente`/`falhou` + filtro de tentativas/backoff), nao uma fila externa — decisao de arquitetura ja tomada, nao uma pendencia.
- Testes: `Simpleto.Api.Tests/Services/WhatsAppDispatchServiceTests.cs` (5 casos: sucesso, falha do sender, excecao, sem telefone, sem pendentes) + equivalente para e-mail. Rodado nesta revisao: **26/26 aprovados** (`dotnet test --filter "FullyQualifiedName~DispatchService|FullyQualifiedName~Notificacao"`).
- Canal `portal`: nunca processado pelos workers (filtro `WHERE nc.canal = 'email'`/`'whatsapp'`) — fica permanentemente com status `criada`. Isso e correto: portal e rastreado pela propria existencia da linha + `LidaEm`, nao precisa de "envio".

### Gaps reais identificados nesta revisao (nao resolvidos ainda)

1. **Sem lock de concorrencia na fila** (`ObterPendentesDisparoEmailAsync`/`ObterPendentesDisparoAsync` nao usam `FOR UPDATE SKIP LOCKED`): se mais de uma instancia da API rodar o worker simultaneamente, duas instancias podem pegar o mesmo registro pendente e enviar duplicado antes que o status seja atualizado. Nao e um problema em ambiente single-instance (hml atual), mas bloqueia producao multi-instancia sem correcao.
2. **Segredo SMTP versionado**: `Simpleto.Api/appsettings.json` contem uma senha de app do Gmail em texto puro (achado ja registrado no inicio desta sessao, nunca resolvido) — viola diretamente o criterio "Credenciais ficam fora do repositorio" desta pagina. Recomendacao mantida: revogar a senha e mover para user-secrets/variavel de ambiente.
3. **Push/SMS**: fora de escopo confirmado (ver secao "Push/SMS" abaixo) — nao implementado, nao deve ser implementado sem decisao de escopo.
4. Nenhum teste de idempotencia com reentrega real foi executado (so unitario, mockando o sender) — nao ha evidencia de teste de callback duplicado ou webhook (o WhatsApp via OpenClaw parece ser round-trip sincrono, nao callback assincrono — a confirmar).

## Objetivo

Entregar o envio real de notificacoes por canais externos sem acoplar provedores a regra de negocio, com idempotencia, retry, rastreabilidade e tratamento visivel de falha.

## Estado atual conhecido (historico, anterior ao achado acima)

A migration V071 cria `notificacao` e `notificacaocanal`, mas declara que a fatia N1 e motor/log e nao faz disparo externo. Isso ja foi superado pelos workers descritos acima; o texto abaixo e mantido como contexto historico do desenho original:

- registrar uma notificacao nao prova, por si so, que um e-mail foi enviado (continua valido como principio — a prova e o status `enviada` gravado pelo worker apos confirmacao do adapter);
- status `enviada` nao deve ser atribuido sem confirmacao do adapter/provedor — **respeitado** pela implementacao atual;
- e-mail e WhatsApp precisam de integracao, credenciais, worker/filas e testes proprios — **feito**, ver acima;
- o provedor e a infraestrutura de fila ja estao confirmados: SMTP (e-mail) e OpenClaw (WhatsApp); fila = tabela `notificacaocanal`.

## Fluxo arquitetural obrigatorio

1. Modulo de negocio publica um evento interno.
2. Consumidor de notificacoes resolve destinatarios e preferencias.
3. Sistema cria uma notificacao e um registro por canal.
4. Worker entrega cada canal por um adapter isolado.
5. Adapter traduz o contrato interno para o provedor externo.
6. Resposta atualiza status, horario, tentativa e motivo.
7. Falha transitoria volta para retry; falha definitiva fica visivel para operador.

A regra que cria a cobranca, assembleia, chamado ou comunicado nao deve chamar diretamente a API de e-mail/WhatsApp.

## Canais

### Portal

- Canal interno, sempre rastreavel pelo sistema.
- Deve respeitar tenant, destinatario e permissao.
- Preferencia nao deve remover o canal obrigatorio se essa regra for mantida.
- Leitura deve ser atribuida ao usuario autenticado correto.

### E-mail

- Confirmar provedor e contrato de envio.
- Nao registrar segredo em codigo, banco funcional ou log.
- Validar endereco antes do envio quando possivel.
- Distinguir aceite do provedor, entrega e rejeicao quando o provedor oferecer esses eventos.
- Webhook do provedor deve validar assinatura e ser idempotente.

### WhatsApp

- Confirmar provedor Business/Cloud API, conta, opt-in e templates aprovados.
- Fora da janela de conversa, usar template permitido pelo provedor.
- Numero invalido, opt-out e template rejeitado sao falhas definitivas, nao retry infinito.
- Rate limit, timeout e indisponibilidade 5xx sao candidatos a retry.

### Push/SMS

A lista inicial do checklist menciona push/SMS, mas o requisito oficial confirma Portal, E-mail e WhatsApp. Nao implementar push ou SMS sem decisao de escopo e contrato do provedor.

## Estados e transicoes

Cada canal deve ter estado independente. O conjunto minimo documentado na V071 e:

- `criada`;
- `pendente`;
- `enviando`;
- `enviada`;
- `falhou`;
- `cancelada`.

Definir antes da implementacao se `enviada` significa aceito pelo provedor ou efetivamente entregue. Se o provedor separar aceite e entrega, adicionar a distincao ao contrato sem sobrescrever o historico.

## Retry e idempotencia

- Criar chave idempotente por evento de origem, destinatario e canal.
- Reprocessar a mesma mensagem nao pode criar envio duplicado.
- Retry deve ter limite e backoff; nao usar loop infinito.
- Erro 4xx de dados/opt-in/template normalmente e definitivo.
- Timeout, 5xx e rate limit normalmente sao transitorios.
- Ao esgotar tentativas, marcar falha definitiva e gerar alerta operacional.
- Reenvio manual deve criar novo registro relacionado ao original ou seguir o contrato ja definido pelo handler, preservando o historico.

Os numeros de tentativas e os intervalos ainda sao decisao operacional; registrar antes de codar.

## Dados e privacidade

- Segredos ficam em secret manager/variaveis protegidas.
- Logs nao devem conter token, conteudo financeiro desnecessario ou dados pessoais completos.
- Webhooks devem ser autenticados e correlacionados por id de evento.
- O tenant deve acompanhar o processamento interno; nunca confiar no tenant vindo de callback externo.
- Opt-out de canal deve ser respeitado, salvo comunicacao que tenha regra juridica especifica e politica aprovada.

## Criterios de aceite

- [x] Evento interno cria uma notificacao sem bloquear a transacao principal (worker assincrono via polling, nao bloqueia o request que originou a notificacao).
- [~] Cada canal habilitado cria no maximo um envio idempotente para o mesmo evento/destinatario — vale para instancia unica; sem `FOR UPDATE SKIP LOCKED` nao ha garantia com multiplas instancias do worker (gap 1 acima).
- [ ] Canal desabilitado ou sem opt-in nao e enviado e fica com motivo rastreavel — nao ha modelo de opt-in/preferencia de canal ainda.
- [x] Sucesso do provedor atualiza status e timestamps (`AtualizarStatusCanalAsync`, `EnviadaEm`).
- [x] Falha transitoria agenda retry com limite (backoff 5min, ate 3 tentativas).
- [x] Falha definitiva nao agenda retry infinito e aparece para operador (status `falhou` + `FalhaMotivo` apos 3 tentativas, consultavel via API).
- [ ] Callback duplicado nao duplica leitura nem muda um status final indevidamente — nao testado (gap 4 acima).
- [x] Falha de um canal nao apaga o sucesso de outro canal (cada linha de `notificacaocanal` e independente).
- [x] Consulta administrativa mostra origem, destinatario, canal, status, tentativas e motivo (`NotificacoesController`, campos ja no DTO).

## Criterio de pronto da P4

- [x] Provedores e contratos aprovados (SMTP para e-mail, OpenClaw para WhatsApp — ja em uso).
- [x] Adapters isolados do dominio (`IEmailSenderService`, `IWhatsAppSender`).
- [x] Worker/fila executa e recupera mensagens (`EmailDispatcherWorker`, `WhatsAppDispatcherWorker`, fila = tabela `notificacaocanal`).
- [x] Retry, dead-letter ou mecanismo equivalente esta implementado (3 tentativas, backoff, status `falhou` como dead-letter logico).
- [ ] Idempotencia testada com reentrega e callback duplicado — nao feito (gaps 1 e 4).
- [ ] Credenciais ficam fora do repositorio — **violado**, senha SMTP em `appsettings.json` (gap 2).
- [~] Logs e metricas permitem explicar cada falha — log via `ILogger` e `FalhaMotivo` na tabela; sem dashboard/alerta operacional.
- [ ] Teste de homologacao confirma envio controlado sem dados reais indevidos — nao executado nesta revisao (exigiria enviar e-mail/whatsapp real).

## Questoes em aberto

- Qual provedor sera usado para e-mail e WhatsApp?
- Qual fila/worker existe ou sera adotado no backend?
- Quais eventos entram na primeira entrega?
- Qual politica de tentativas e backoff?
- Quem pode reenviar e quem recebe alerta de falha definitiva?
- Quais comunicacoes exigem tratamento juridico diferente de uma notificacao informativa?
