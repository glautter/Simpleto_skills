# Guia de continuidade: Comunicados e Notificacoes

Data de atualizacao: 2026-09-27

## Objetivo deste conjunto de documentos

Este diretorio permite que outra pessoa ou outro modelo continue o trabalho sem depender do historico da conversa. Cada etapa deve registrar:

- o objetivo de negocio;
- o estado real confirmado no repositorio;
- os arquivos que sao fonte de verdade;
- o que esta implementado;
- o que ainda falta;
- a validacao que prova a conclusao;
- as decisoes que ainda precisam de resposta humana.

## Ordem de leitura

1. [CHECKLIST_IMPLEMENTACAO.md](../../CHECKLIST_IMPLEMENTACAO.md): indice geral e proximo passo.
2. [P1-BASE-COMUNICADOS-NOTIFICACOES.md](P1-BASE-COMUNICADOS-NOTIFICACOES.md): estado tecnico atual e pendencias imediatas.
3. [P2-GESTAO-ADMINISTRATIVA.md](P2-GESTAO-ADMINISTRATIVA.md): proxima fatia depois de fechar a validacao da P1.
4. [P3-EXPERIENCIA-USUARIO-FINAL.md](P3-EXPERIENCIA-USUARIO-FINAL.md): jornada e criterios do usuario final.
5. [P4-INTEGRACOES-CANAIS.md](P4-INTEGRACOES-CANAIS.md): canais externos, retry, idempotencia e falhas.
6. [P5-QUALIDADE-SEGURANCA-OPERACAO.md](P5-QUALIDADE-SEGURANCA-OPERACAO.md): testes, seguranca, observabilidade e release.
7. Requisitos oficiais do backend:
   - [modelo conceitual](../../Simpleto.BackApi/docs/requirements/REQ-20260723-comunicados-notificacoes-modelo-conceitual.md)
   - [epico e historias](../../Simpleto.BackApi/docs/requirements/REQ-20260723-comunicados-notificacoes-epico-historias.md)

## Estado em 2026-09-27

| Prioridade | Estado | Significado |
|---|---|---|
| P0 | Concluida | Indicador de versao e release notes no header do frontend; build Angular anterior passou. |
| P1 | Em andamento | O backend possui as fatias C1 e N1, mas a validacao completa e a integracao das telas ainda nao estao comprovadas neste documento. |
| P2 | Nao iniciada como entrega | O escopo funcional esta definido, mas a tela administrativa e o fluxo operacional precisam ser confirmados no frontend. |
| P3 | Documentada; implementacao nao iniciada | Experiencia do usuario final depende da P1 e da decisao sobre o portal/app alvo. |
| P4 | Parcial no desenho | Portal, e-mail e WhatsApp estao previstos; a migration V071 declara que a fatia N1 nao faz disparo externo. |
| P5 | Documentada; validacao nao concluida | Testes, seguranca, observabilidade e procedimento de release precisam de evidencias recentes. |

## Como interpretar os status

- `Concluida`: existe implementacao e existe validacao registrada.
- `Em andamento`: parte existe, mas ainda ha pendencias que impedem declarar a fatia pronta.
- `Nao iniciada`: ainda nao ha implementacao confirmada para o objetivo.
- `Decisao pendente`: nao codificar por suposicao; registrar a pergunta e obter resposta.

## Regra de continuidade

Antes de editar:

1. leia o checklist e o documento da prioridade;
2. abra os arquivos listados como fonte de verdade;
3. confirme se o estado descrito ainda e verdadeiro;
4. execute a validacao barata indicada;
5. faca a menor alteracao que fecha um item;
6. rode a validacao novamente;
7. atualize o documento e o checklist com data, arquivos e comando.

## Regra de evidencia

Uma caixa so pode ser marcada como concluida quando houver uma destas evidencias, conforme o item:

- teste automatizado aprovado;
- build/compilacao aprovada;
- migration aplicada e confirmada no ambiente alvo;
- fluxo HTTP testado com autorizacao e isolamento de tenant;
- revisao visual/funcional confirmada no frontend;
- decisao de negocio registrada em documento.

Nao confundir existencia de arquivo com funcionalidade pronta. Uma entidade, controller ou migration pode existir e ainda estar sem registro, sem teste, sem tela ou sem disparo externo.

## Fontes de verdade tecnicas confirmadas

- Comunicado: `Simpleto.BackApi/Simpleto.Domain/Entities/Comunicado.cs`
- Notificacao: `Simpleto.BackApi/Simpleto.Domain/Entities/Notificacao.cs`
- Tabelas C1/N1: `Simpleto.BackApi/Simpleto.Infrastructure/Migrations/V071__comunicado_notificacao_tables.sql`
- Regra de aviso sobre convocacao: `Simpleto.BackApi/Simpleto.Infrastructure/Migrations/V072__comunicado_convocacao_assembleia.sql`
- API de comunicados: `Simpleto.BackApi/Simpleto.Api/Controllers/v1/NoticiasController.cs`, rota `/api/v1/comunicados`
- API de notificacoes: `Simpleto.BackApi/Simpleto.Api/Controllers/v1/NotificacoesController.cs`, rota `/api/v1/notificacoes`
- Requisitos: `Simpleto.BackApi/docs/requirements/REQ-20260723-comunicados-notificacoes-epico-historias.md`

## Proximo passo sem ambiguidade

Fechar P1 na seguinte ordem:

1. executar build e testes do backend;
2. confirmar que as migrations V071-V074 foram aplicadas no ambiente de desenvolvimento/homologacao alvo;
3. testar os endpoints com usuario autenticado, permissao correta, permissao ausente e tenant incorreto;
4. localizar no frontend os consumidores de `/api/v1/comunicados` e `/api/v1/notificacoes`;
5. implementar ou registrar como P2 as telas que ainda estiverem ausentes;
6. atualizar as caixas do checklist somente com as evidencias obtidas.
