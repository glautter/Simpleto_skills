# P4 - Integracoes de canais e notificacoes

Data de atualizacao: 2026-09-27
Status: DESENHO PARCIAL; IMPLEMENTACAO EXTERNA NAO CONFIRMADA

## Objetivo

Entregar o envio real de notificacoes por canais externos sem acoplar provedores a regra de negocio, com idempotencia, retry, rastreabilidade e tratamento visivel de falha.

## Estado atual conhecido

A migration V071 cria `notificacao` e `notificacaocanal`, mas declara que a fatia N1 e motor/log e nao faz disparo externo. Portanto:

- registrar uma notificacao nao prova que um e-mail foi enviado;
- status `enviada` nao deve ser atribuido sem confirmacao do adapter/provedor;
- e-mail e WhatsApp precisam de integracao, credenciais, worker/filas e testes proprios;
- o provedor e a infraestrutura de fila ainda precisam ser confirmados.

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

- [ ] Evento interno cria uma notificacao sem bloquear a transacao principal.
- [ ] Cada canal habilitado cria no maximo um envio idempotente para o mesmo evento/destinatario.
- [ ] Canal desabilitado ou sem opt-in nao e enviado e fica com motivo rastreavel.
- [ ] Sucesso do provedor atualiza status e timestamps.
- [ ] Falha transitoria agenda retry com limite.
- [ ] Falha definitiva nao agenda retry infinito e aparece para operador.
- [ ] Callback duplicado nao duplica leitura nem muda um status final indevidamente.
- [ ] Falha de um canal nao apaga o sucesso de outro canal.
- [ ] Consulta administrativa mostra origem, destinatario, canal, status, tentativas e motivo.

## Criterio de pronto da P4

- [ ] Provedores e contratos aprovados.
- [ ] Adapters isolados do dominio.
- [ ] Worker/fila executa e recupera mensagens.
- [ ] Retry, dead-letter ou mecanismo equivalente esta implementado.
- [ ] Idempotencia testada com reentrega e callback duplicado.
- [ ] Credenciais ficam fora do repositorio.
- [ ] Logs e metricas permitem explicar cada falha.
- [ ] Teste de homologacao confirma envio controlado sem dados reais indevidos.

## Questoes em aberto

- Qual provedor sera usado para e-mail e WhatsApp?
- Qual fila/worker existe ou sera adotado no backend?
- Quais eventos entram na primeira entrega?
- Qual politica de tentativas e backoff?
- Quem pode reenviar e quem recebe alerta de falha definitiva?
- Quais comunicacoes exigem tratamento juridico diferente de uma notificacao informativa?
