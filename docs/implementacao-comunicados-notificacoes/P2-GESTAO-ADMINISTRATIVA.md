# P2 - Gestao administrativa de Comunicados

Data de atualizacao: 2026-09-27
Status: EM ANDAMENTO — parte substancial ja implementada no frontend (Angular, `Simpleto_front_adm`), mas com gaps reais confirmados por leitura de codigo (nao houve validacao visual/E2E na tela ainda, so revisao estatica). Ver "Estado confirmado no codigo" abaixo.

## Estado confirmado no codigo (2026-09-27)

Revisao estatica dos arquivos em `src/app/pages/condominio/comunicacao/comunicados-gestao/` e
`src/app/core/comunicacao/comunicados.service.ts`, rota `condominio.routes.ts`.

**Implementado:**
- Listagem (`ComunicadosGestaoComponent` / `comunicados-gestao.html`), usando `app-list-base`
  com colunas titulo/canal/status/publicado-em.
- Criacao de comunicado via dialog (`ComunicadoFormDialogComponent` -> `ComunicadoFormComponent`),
  formulario com titulo/resumo/conteudo/canal/segmentacao (condominio, bloco, unidade, perfil) e
  checkbox de "relacionado a convocacao de assembleia". Validacao local via `CrudFormComponent`
  (campos `required` bloqueiam envio).
- Detalhe (`ComunicadoDetalheComponent`), com botoes Publicar/Arquivar condicionados a
  `podePublicar()`/`podeArquivar()` (regra de estado local, espelhando a API), aviso de convocacao
  de assembleia exibido quando `envolveConvocacaoAssembleia` + `avisoConvocatoriaFormal` existem,
  segmentacao exibida, e lista de leituras com contagem (`getLeituras`/`getContagemLeituras`).
- Tratamento de erro basico: toda chamada tem `error` callback chamando
  `notificationService.error(buildApiErrorMessage(...))` — nao mostra "sucesso falso" quando a API
  rejeita.

**Gaps reais confirmados (nao implementado ainda, apesar da tela existir):**
1. **Nao ha UI para editar um rascunho existente.** `ComunicadosService.update()` existe e e usado
   em nenhum lugar do frontend — `ComunicadoDetalheComponent` so tem Publicar/Arquivar, sem botao
   "Editar", e o dialog de formulario (`comunicados-gestao.ts::onAdd`) so e usado para criar. Isso
   quebra o criterio "Ao editar, o id da rota nao pode ser substituido..." (nao aplicavel ainda
   porque a funcionalidade nao existe) e o proprio fluxo principal do documento (passo 6: "usuario
   pode revisar" um rascunho).
2. **Publicar e arquivar nao pedem confirmacao.** `publicar()`/`arquivar()` em
   `comunicado-detalhe.ts` chamam a API direto no `(click)`, sem dialog de confirmacao — o
   documento exige confirmacao antes de publicar e antes de arquivar.
3. **Sem filtros na tela de gestao.** `comunicados-gestao.ts::loadData()` chama
   `comunicadosService.list()` sem nenhum filtro (canal/periodo/status de leitura), e o componente
   nao passa `[filters]` para `app-list-base` (que suporta filtro, ver
   `shared/list-base/list-base.component.ts`) — o filtro simplesmente nao foi configurado nesta
   tela. Contraste: a tela de notificacoes (`notificacoes.ts`) ja tem filtro por canal/status
   funcionando, entao o padrao existe no codebase, so nao foi replicado aqui.
4. **Sem publicacao programada.** O formulario nao tem campo de data/hora futura — bate com a API,
   que tambem nao tem esse conceito (`POST .../publicar` e imediato); e decisao de produto/API em
   aberto, nao so de frontend.
5. **Sem acoes de "cancelar" ou "reabrir".** So existem publicar/arquivar, alinhado ao ciclo de
   vida real da API (`rascunho -> publicado -> arquivado`, sem essas transicoes) — nao e bug, mas
   contradiz o texto do escopo do checklist principal ("Acoes de arquivar, cancelar e reabrir"),
   que parece ter sido escrito antes do modelo de dados existir.
6. **Nao ha tratamento visual diferenciado para 403 vs. outros erros**, nem para "sessao expirada"
   de forma explicita na tela (o redirecionamento em caso de 401 e global, via
   `error.interceptor.ts`, nao uma mensagem inline nesta tela).
7. **Sem tratamento de conflito de concorrencia** (dois usuarios editando/publicando o mesmo
   registro ao mesmo tempo) — nao ha verificacao de versao/ETag no update.

Nenhum destes gaps foi corrigido nesta sessao — ficaram documentados para decisao/priorizacao do
usuario, e a revisao foi so estatica (leitura de codigo), sem rodar a tela no navegador.

## Objetivo

Dar ao sindico, administradora e demais perfis autorizados uma tela operacional para criar, revisar, publicar, arquivar e acompanhar comunicados sem depender de chamadas manuais da API.

P2 so deve comecar depois que P1 estiver validada. Se a API ainda tiver contrato instavel, a tela criara retrabalho e pode mascarar problemas de autorizacao ou tenant.

## Usuarios e permissoes

- Leitor: consulta comunicados permitidos pelo escopo.
- Autor: cria e edita rascunhos conforme `comunicado:create` e `comunicado:update`.
- Gestor: publica, arquiva e consulta leituras conforme `comunicado:manage`.
- Administradora: pode operar multiplos condominios somente dentro do escopo concedido pelo RBAC.

A tela nao deve esconder uma acao apenas por convencao visual; a API continua sendo a autoridade final. A interface deve tratar 401/403 como estados diferentes e apresentar uma mensagem compreensivel.

## Fluxo principal

1. Usuario entra na lista de comunicados.
2. Sistema carrega pagina, filtros e estado de leitura/status.
3. Usuario abre um rascunho existente ou inicia novo.
4. Usuario informa titulo, resumo, conteudo, canal e publico-alvo.
5. Sistema valida campos localmente e envia para a API.
6. Rascunho e salvo; usuario pode revisar.
7. Usuario com permissao de gestao publica.
8. Sistema mostra status, data e autor.
9. Usuario pode arquivar quando a comunicacao deixar de ser operacional.
10. Gestor consulta contagem/lista de leituras quando essa informacao estiver autorizada.

## Tela de listagem

Deve conter:

- status: rascunho, publicado, arquivado;
- canal: geral, financeiro, manutencao, seguranca;
- periodo de publicacao;
- status de leitura quando a visao for do proprio usuario;
- titulo/resumo;
- autor e data;
- acoes condicionadas a permissao: editar, publicar, arquivar, abrir leituras.

Estados obrigatorios:

- carregando;
- lista vazia;
- erro de rede;
- sessao expirada;
- sem permissao;
- erro de validacao retornado pela API;
- sucesso de cada mutacao;
- tentativa de publicar enquanto outro usuario alterou o registro.

## Formulario de criacao/edicao

Campos a confirmar com o DTO real antes de codar:

- titulo: obrigatorio, limite conforme contrato da API;
- resumo: obrigatorio;
- conteudo: opcional conforme entidade, mas a regra de produto deve ser clara;
- canal tematico;
- caminho da imagem, se a tela suportar imagem;
- publico-alvo por condominio, bloco, unidade ou perfil;
- indicador de relacao com convocacao de assembleia.

O formulario deve explicar que um comunicado relacionado a assembleia nao substitui a convocacao formal. Nao usar texto que sugira ciencia juridica pelo simples ato de leitura.

## Criterios de aceite

### Criar e editar

- [x] Com permissao de criacao, o usuario consegue salvar um rascunho valido (`ComunicadoFormComponent` + `ComunicadosService.create`).
- [x] Campos invalidos impedem o envio e mostram o erro no campo correspondente (`CrudFormComponent`, campos `required`).
- [ ] Ao editar, o id da rota nao pode ser substituido por id informado em outro campo — **N/A, edicao nao existe na UI ainda** (gap 1 acima).
- [x] Sem permissao, a API rejeita a operacao e a tela nao informa sucesso falso (todo `subscribe` tem `error` callback com `notificationService.error`, nenhum assume sucesso).
- [x] Apos salvar, a tela mostra o status real retornado pela API (`loadData()`/`loadAll()` recarrega da API apos cada mutacao, nao atualiza estado local otimisticamente).

### Publicar e arquivar

- [x] Publicar exige permissao de gestao (rota `comunicados-gestao/:id` com `permission: { recurso: 'comunicado', acao: 'manage' }`; API tambem exige `comunicado:manage`).
- [ ] A tela pede confirmacao antes da publicacao — **gap 2 acima, nao implementado.**
- [x] Apos publicar, o botao de editar respeita a regra real do backend — **N/A, nao ha botao de editar ainda**, mas os botoes Publicar/Arquivar seguem `podePublicar()`/`podeArquivar()` corretamente.
- [ ] Arquivar exige confirmacao e atualiza a lista sem apagar o historico — confirmacao **nao implementada** (gap 2); atualizacao sem apagar historico esta ok (arquivar so muda status).
- [x] O aviso de convocacao aparece quando aplicavel (`comunicado-detalhe.html`, bloco `@if (comunicado.envolveConvocacaoAssembleia && comunicado.avisoConvocatoriaFormal)`).

### Filtros e leitura

- [ ] Os filtros sao enviados nos nomes exatos aceitos pela API — **N/A, filtros nao existem nesta tela** (gap 3 acima; o service `ComunicadosService.list()` aceita filtro, so nao e usado pela tela de gestao).
- [x] A lista nao exibe comunicados fora do tenant ou do escopo do usuario (API aplica RLS por tenant; tela so exibe o que a API retorna, sem filtro client-side que pudesse vazar tenant).
- [x] Marcar como lido atualiza a linha/detalhe sem criar duplicidade (validado via HTTP em P1.4: `contagem-leituras` nao duplicou; a tela chama o mesmo endpoint).
- [x] A consulta de leituras de terceiros so aparece para quem possui gestao (rota `comunicados-gestao/:id` exige `comunicado:manage`; API tambem exige `comunicado:manage` no endpoint `/leituras`).

## Dependencias

- P1 validada e endpoints estaveis.
- Modelos TypeScript alinhados aos DTOs reais.
- Interceptor de tenant e autenticacao funcionando.
- Padrao visual existente do frontend Angular.
- Decisao sobre paginação e ordenacao da API.

## Nao fazer nesta etapa

- Nao criar uma nova entidade de frontend que contradiga `Comunicado`.
- Nao chamar a entidade legada `Noticia` como se fosse a fonte de verdade nova.
- Nao implementar envio de WhatsApp dentro da tela de publicacao.
- Nao tratar leitura como aceite juridico.
- Nao liberar acao administrativa apenas porque o usuario consegue ver o botao.

## Criterio de pronto da P2

- [ ] Lista, filtros, formulario, publicacao e arquivamento funcionam contra a API real — **filtros e edicao faltam** (gaps 1 e 3); resto existe no codigo mas nao foi exercitado no navegador nesta sessao.
- [ ] Estados de erro e permissao foram testados — **nao testado no navegador**, so revisao estatica.
- [ ] Segmentacao foi exercitada com pelo menos dois escopos — **nao testado no navegador** (a API foi testada com segmentacao em P1.4, a tela nao).
- [ ] A tela funciona em desktop e viewport menor sem cortar campos ou acoes — **nao verificado**.
- [ ] Testes frontend relevantes passam — **nao ha testes automatizados de frontend identificados para este modulo**; nao foi executado `ng test`.
- [x] Checklist principal registra arquivos alterados e comando de validacao (esta revisao).

P2 continua **em andamento**, nao pronta. Pendente decisao do usuario sobre prioridade dos gaps 1-7 acima antes de considerar a fatia fechada.

## Proximo documento a criar depois da P2

Quando P2 estiver validada, criar `P3-EXPERIENCIA-USUARIO-FINAL.md` com a jornada do morador, sino de notificacoes, detalhe do comunicado, leitura e acessibilidade. Nao antecipar esse documento como implementacao sem confirmar se o portal/app do morador esta neste mesmo workspace.
