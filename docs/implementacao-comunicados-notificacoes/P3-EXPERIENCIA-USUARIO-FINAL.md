# P3 - Experiencia do usuario final

Data de atualizacao: 2026-09-27
Status: EM ANDAMENTO — parte substancial ja implementada (`ComunicadosLeituraComponent` e `NotificacoesComponent`, ambos reaproveitados nos portais `condominio` e `morador` do mesmo app Angular). Gaps de UI e um **achado de seguranca no backend** (autorizacao ausente em 2 endpoints) documentados abaixo — ver "Achado de seguranca".

## Objetivo

Permitir que o morador ou usuario final encontre, entenda e acompanhe comunicados e notificacoes no canal destinado a ele, com linguagem clara, poucos passos e sem confundir leitura com ciencia formal.

## Decisao de produto necessaria antes de codar

[RESOLVIDO 2026-09-27, por evidencia de codigo, nao suposicao] A experiencia do morador **e uma
area no mesmo frontend administrativo** (`Simpleto_front_adm`), em `src/app/portals/morador/`,
mesmo padrao ja usado para dezenas de outras telas do morador (financeiro, reservas, chamados,
anuncios, assembleias etc. — ver `morador.routes.ts`). Os comunicados/notificacoes ja seguem esse
padrao: `ComunicadosLeituraComponent` e `NotificacoesComponent` sao registrados tanto em
`portals/condominio/condominio.routes.ts` quanto em `portals/morador/morador.routes.ts` (mesmo
componente, reaproveitado nos dois portais). Nao ha app separado nem outro frontend para isso.

## Estado confirmado no codigo (2026-09-27)

- **Lista de comunicados** (`ComunicadosLeituraComponent`): filtro por canal, filtro "apenas nao
  lidos", busca por titulo/resumo, cards de total/lidos/nao-lidos, botao "Confirmar leitura" com
  dialog de confirmacao (`notificationService.confirmYesNo`) antes de chamar a API — atende boa
  parte do criterio "Lista" e "Leitura".
- **Notificacoes** (`NotificacoesComponent`): filtro por canal/status, preferencias de canal do
  usuario (portal nao pode ser desabilitado, conforme regra do documento), botao "Reenviar" para
  canais com status `falhou`.

## Achado de seguranca (2026-09-27) — decisao pendente, nao corrigido

Revisando o fluxo de "Reenviar" exposto ao morador, encontrei que **2 dos 5 endpoints de canal de
notificacao nao verificam posse (ownership)** antes de agir — diferente do quinto, que faz isso
corretamente e ate documenta o motivo:

- `PUT /api/v1/notificacoes/canais/{id}/status` (`AtualizarStatusNotificacaoCanalHandler`)
- `POST /api/v1/notificacoes/canais/{id}/reenviar` (`ReenviarNotificacaoCanalHandler`)

Os dois so validam `[Authorize]` (autenticado) + `TenantId` (via
`INotificacaoRepository.ObterContextoCanalAsync(tenantId, notificacaoCanalId)`, que filtra so por
tenant). **Nenhum dos dois confere se `contexto.DestinatarioId == _userContext.UserId`.**

Por contraste, `MarcarNotificacaoCanalComoLidaHandler` (`PUT .../lida`) faz exatamente essa
checagem e tem um comentario XML explicito: *"Diferente dos handlers vizinhos de canal
(Reenviar/AtualizarStatus), este precisa checar posse — qualquer usuario autenticado nao pode
marcar notificacao alheia como lida."* — ou seja, o padrao correto ja existe no codebase ao lado
do codigo com a lacuna.

**Impacto concreto:** qualquer usuario autenticado do tenant que souber (ou obtiver de algum jeito)
o `notificacaoCanalId` de OUTRO usuario pode:
- reenviar a notificacao de outra pessoa (`POST .../reenviar`) — a resposta desse endpoint ainda
  devolve `Titulo`/`Corpo` da notificacao alheia, um vazamento de conteudo além da acao em si;
- alterar o status/motivo de falha de uma notificacao alheia (`PUT .../status`) — pode marcar como
  "enviada" ou "falhou" com um motivo arbitrario um registro que nao e dele.

Isso bate diretamente com o criterio deste documento ("Reenvio e acao administrativa, nao uma acao
disponivel para qualquer morador") e com a premissa geral de isolamento por usuario. Nao e um
problema de isolamento de tenant (RLS/tenant_id continuam corretos) — e um problema de autorizacao
por registro (IDOR) dentro do mesmo tenant.

**Por que nao corrigi sem perguntar:** a correcao certa depende de uma decisao de produto que nao
esta escrita em lugar nenhum do repositorio:
1. `PUT .../status` e descrito no proprio comentario do handler como "chamado pelo adapter de
   canal" — ou seja, parece pensado para ser uma chamada *backend-to-backend* (o worker de
   disparo real, quando existir em P4), nao uma chamada de usuario final. Hoje nao existe uma
   permissao RBAC `notificacao:*` no sistema (conferido na lista de permissions do JWT de teste em
   P1.3/P1.4 — so existe `comunicado:*`), entao restringir esse endpoint exigiria criar uma
   permissao nova ou um mecanismo de autenticacao servico-a-servico, o que e decisao de
   arquitetura, nao correcao pontual.
2. `POST .../reenviar` pode fazer sentido tanto como autoatendimento do proprio destinatario
   (adicionar checagem de posse, igual `MarcarComoLida`) quanto como acao administrativa (exigir
   uma permissao nova de gestao) — o documento sugere a segunda leitura, mas isso muda quem pode
   usar o botao "Reenviar" que hoje aparece pra qualquer usuario (inclusive morador) no
   `NotificacoesComponent`.

Nenhuma alteracao de codigo foi feita para este achado. Fica registrado como pendencia de alta
prioridade para decisao do usuario antes de considerar P3 (ou mesmo P1, ja que o endpoint e do
nucleo N1) pronta para producao.

## Perfil e contexto otimizado

- Morador/condomino: uso esporadico, frequentemente no celular, precisa localizar uma informacao sem conhecer a estrutura interna do sistema.
- Conselho: consulta ocasional, precisa distinguir comunicados gerais de informacoes administrativas.
- Usuario administrativo: precisa visualizar a experiencia final para verificar alcance, mas nao deve receber a mesma navegacao simplificada do morador por acidente.

## Jornada principal do morador

1. O usuario entra no portal e identifica que existem comunicados nao lidos.
2. Abre a lista de comunicados.
3. Ve primeiro os itens relevantes e recentes, sem perder acesso ao historico permitido.
4. Filtra por canal/categoria quando necessario.
5. Abre o detalhe.
6. Le o titulo, resumo, conteudo, data e origem.
7. O sistema registra a leitura do usuario autenticado.
8. O usuario retorna a lista e ve o estado atualizado.

## Informacao minima da lista

Cada item deve permitir entender sem abrir varias telas:

- titulo;
- resumo curto;
- canal/categoria;
- data de publicacao;
- indicador de nao lido;
- prioridade, somente se existir campo/regra confirmada;
- identificacao de comunicado arquivado, quando o historico for exibido.

## Detalhe do comunicado

Deve apresentar:

- titulo e data;
- conteudo completo;
- imagem, se houver e se o canal suportar;
- aviso explicito quando envolver convocacao de assembleia, informando que o comunicado nao substitui a convocacao formal;
- estado de leitura sem linguagem de aceite juridico;
- acao de voltar sem perder filtros e posicao da lista.

**Gap confirmado (2026-09-27): esta tela nao existe para o morador.** `ComunicadosLeituraComponent`
so mostra `titulo`/`resumo` na lista e um botao "Confirmar leitura" — nao ha rota `comunicados/:id`
nem qualquer forma de o morador ver `conteudo` completo, `avisoConvocatoriaFormal` ou imagem. O
`ComunicadoDetalheComponent` que existe (`comunicados-gestao/comunicado-detalhe.ts`) e exclusivo do
portal `condominio`, exige permissao `comunicado:manage` e mostra a lista de leituras — nao serve
para o morador (mostraria dados administrativos que o morador nao deveria ver, e a rota nem esta
registrada em `morador.routes.ts`). Precisa de um componente de detalhe novo, so-leitura, sem a
secao de leituras administrativas.

## Notificacoes

A experiencia deve diferenciar:

- comunicado: conteudo editorial que pode ser lido no portal;
- notificacao: aviso originado por evento e entregue por canal;
- documento financeiro: documento individualizado, que nao deve aparecer como comunicado generico sem permissao e regra propria.

O sino/central de notificacoes deve informar canal, data, status e origem de forma compreensivel. Nao exibir como entregue algo que a API marcou como falho.

## Criterios de aceite

### Lista

- [x] Usuario autenticado ve apenas itens do seu tenant e escopo (API aplica RLS por tenant; sem filtro client-side que vaze).
- [x] Usuario ve diferenca clara entre lido e nao lido (coluna `leituraTxt` + cards de contagem em `comunicados-leitura.html`).
- [~] Lista vazia tem mensagem orientativa — depende do comportamento generico do `app-list-base` para lista vazia; **nao verificado visualmente**.
- [x] Erro de rede permite tentar novamente — `loadData()` so desliga `loading`, o usuario pode acionar de novo (nao ha bloqueio permanente), mas nao ha botao explicito de "tentar novamente".
- [x] Sessao expirada encaminha para autenticacao sem perder informacao sensivel na URL (`error.interceptor.ts`, tratamento global de 401 ja validado em P1.3).
- [ ] Paginacao/rolagem nao duplica itens nem perde filtros — **N/A por enquanto, nao ha paginacao/rolagem infinita, so lista completa client-side**; se o volume crescer isso pode virar problema de performance, nao de duplicacao.

### Leitura

- [~] Abrir detalhe registra leitura para o usuario correto — **nao ha "abrir detalhe"** (gap acima); a leitura e confirmada direto na lista via `marcarComoLido`, que registra o usuario correto (validado via HTTP em P1.4).
- [x] Reabrir nao cria duplicidade (validado via HTTP em P1.4 — `contagem-leituras` nao duplicou ao confirmar leitura duas vezes).
- [x] A tela nao afirma que leitura equivale a ciencia formal (texto e "Confirmar leitura", sem linguagem juridica).
- [x] Um morador nao consegue consultar a leitura de outro morador (endpoint `/leituras` exige `comunicado:manage`; a tela do morador nem chama esse endpoint).

### Notificacao

- [x] Portal mostra somente notificacoes destinadas ao usuario (`GET /notificacoes/minhas` filtra por `DestinatarioId` do usuario autenticado no backend).
- [x] Status de falha e pendencia e compreensivel (`STATUS_NOTIFICACAO_CANAL_LABELS`, coluna dedicada).
- [x] Marcar como lida altera apenas o registro do usuario autenticado (`MarcarNotificacaoCanalComoLidaHandler` confere `DestinatarioId == UserId`, ver P1.4).
- [ ] Reenvio e acao administrativa, nao uma acao disponivel para qualquer morador — **NAO ATENDIDO.** Ver "Achado de seguranca" acima: nem a API nem a tela restringem quem pode reenviar; qualquer usuario autenticado do tenant ve e pode acionar o botao "Reenviar" para qualquer canal com status `falhou`, inclusive de outro usuario.

### Acessibilidade e clareza

- [ ] Fluxo funciona por teclado e leitor de tela conforme o Design System existente — **nao verificado** (precisa de teste manual/axe, nao so leitura de codigo).
- [x] Textos evitam jargao tecnico e juridico desnecessario ("Confirmar leitura", "Nao lido", sem termos juridicos).
- [x] Acoes importantes tem nome claro e confirmacao quando houver efeito de estado (leitura tem confirmacao; reenviar nao tem dialog de confirmacao, mas e uma acao de baixo risco/reversivel).
- [ ] Conteudo e controles cabem em viewport mobile sem sobreposicao — **nao verificado no navegador**; `comunicados-leitura.html` usa `style` inline com `grid-template-columns: repeat(auto-fit, minmax(...))`, o que sugere responsividade, mas nao foi testado.

## Dependencias

- P1 validada e contrato de API confirmado.
- Definicao do frontend/portal alvo.
- Modelos e estados de erro documentados.
- P2 ou endpoints administrativos separados do fluxo do morador.
- Revisao visual pela especialista de UI depois da definicao do fluxo.

## Criterio de pronto da P3

- [ ] Jornada de lista, detalhe e leitura funciona no canal escolhido — **falta a tela de detalhe** (gap confirmado).
- [~] Isolamento de tenant e autorizacao foram testados — tenant ok; **autorizacao por usuario (ownership) falhou** para reenviar/atualizar-status (achado de seguranca).
- [ ] Estados vazio, carregando, erro, nao lido, lido e falha foram validados — parcial, ver checkboxes acima; falta teste manual no navegador.
- [ ] Responsividade e acessibilidade foram verificadas em desktop e mobile — nao verificado.
- [ ] Testes frontend e pelo menos um fluxo integrado passam — nao ha testes automatizados identificados para este modulo.
- [x] Checklist registra o frontend correto, arquivos alterados e comando de validacao (esta revisao).

P3 nao esta pronta. O achado de seguranca (reenviar/atualizar-status sem checagem de posse) e a
ausencia de tela de detalhe do comunicado sao os 2 itens que mais pesam contra o criterio de
pronto.

## Questoes em aberto

- [RESOLVIDO 2026-09-27] Qual projeto/frontend hospeda a experiencia do morador? -> O mesmo app (`Simpleto_front_adm`), portal `morador`, ja em uso para o resto do sistema.
- Existe prioridade entre comunicados e notificacoes no primeiro acesso?
- O usuario pode arquivar ou apenas marcar como lido? -> Pelo estado atual do backend, so marcar como lido; arquivar e acao de gestao (`comunicado:manage`), nao do morador.
- Existe prioridade formal ou apenas data/categoria? -> Nao ha campo de prioridade na entidade `Comunicado` (confirmado em P1) nem na tela.
- O historico tem politica de retencao diferente da comunicacao ativa? -> Sem decisao registrada.
- **[NOVO 2026-09-27] Reenviar/atualizar-status de notificacao devem exigir posse (destinatario) como `MarcarComoLida`, ou virar acao administrativa com permissao nova?** Ver "Achado de seguranca" acima — decisao humana pendente antes de corrigir.
