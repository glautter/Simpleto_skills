# P1 - Base de Comunicados e Notificacoes

Data de atualizacao: 2026-09-27
Status: EM ANDAMENTO / IMPLEMENTACAO PARCIAL

## Objetivo

Entregar a base persistente e protegida do dominio de comunicados e notificacoes. Esta etapa nao esta concluida apenas porque existem tabelas e controllers: a API precisa ser compilada, testada, executada contra um banco com as migrations aplicadas e consumida pelo frontend correto.

## Escopo desta etapa

Incluido:

- Comunicado editorial com ciclo rascunho, publicado e arquivado.
- Segmentacao por condominio, bloco, unidade ou perfil.
- Leitura de comunicado por usuario.
- Notificacao generica ligada a uma origem e a um destinatario.
- Registro de canais por notificacao, com status, tentativas, envio, falha e leitura.
- Configuracao de canais por condominio e preferencia de canal do usuario, conforme migrations posteriores.
- Isolamento por tenant e auditoria estrutural quando o mecanismo de auditoria estiver disponivel.
- API REST v1 usando commands, queries, handlers e repositories existentes.

Fora desta etapa:

- Tela completa de gestao administrativa.
- Portal completo do morador.
- Disparo real de e-mail ou WhatsApp, salvo implementacao posterior explicitamente validada.
- Convocacao formal de assembleia.
- Documentos financeiros com valor probatorio.

## Estado confirmado no codigo

### Comunicado

A entidade `Comunicado` ja existe em `Simpleto.Domain/Entities/Comunicado.cs` com:

- `Id`, `TenantId`;
- `Titulo`, `Resumo`, `Conteudo`;
- `Canal` e `CaminhoImagem`;
- `Status` e `PublicadoEm`;
- `PublicadoPorId`;
- `EnvolveConvocacaoAssembleia`;
- campos de criacao, atualizacao, exclusao logica e versao.

Os valores de canal confirmados sao `geral`, `financeiro`, `manutencao` e `seguranca`. Os estados confirmados sao `rascunho`, `publicado` e `arquivado`.

### Notificacao

A entidade `Notificacao` ja existe em `Simpleto.Domain/Entities/Notificacao.cs` com:

- `Id`, `TenantId`;
- `OrigemTipo` e `OrigemId`;
- `DestinatarioId`;
- `Titulo`, `Corpo` e `CriadaEm`;
- campos de criacao e atualizacao.

O vinculo com cada canal fica em `NotificacaoCanal`, e nao no registro principal da notificacao. Isso permite que um canal falhe sem apagar ou invalidar a intencao de notificar.

### Persistencia

A migration principal e `Simpleto.Infrastructure/Migrations/V071__comunicado_notificacao_tables.sql`. Ela cria:

| Tabela | Responsabilidade |
|---|---|
| `comunicado` | Conteudo e ciclo de vida do comunicado. |
| `comunicadosegmento` | Publico-alvo por uma dimensao de segmentacao por registro. |
| `leituracomunicado` | Uma leitura por usuario e comunicado. |
| `notificacao` | Intencao de notificar um destinatario a partir de uma origem. |
| `notificacaocanal` | Status e rastreio individual de cada canal. |

A migration tambem cria indices, restricoes de canal/status, unicidade de leitura e unicidade de canal por notificacao, RLS por `tenantid` e triggers de auditoria quando as funcoes existirem.

`V072__comunicado_convocacao_assembleia.sql` adiciona `envolveconvocacaoassembleia`. As migrations `V073` e `V074` tratam configuracao de canal por condominio e preferencia por usuario; confirmar o conteudo e a aplicacao delas no ambiente alvo antes de usar como evidencia de pronto.

## API confirmada

### `/api/v1/comunicados`

Controller: `Simpleto.Api/Controllers/v1/NoticiasController.cs`.

| Metodo | Rota | Funcao | Permissao declarada |
|---|---|---|---|
| GET | `/api/v1/comunicados` | Lista comunicados visiveis com filtros | `comunicado:read` |
| GET | `/api/v1/comunicados/{id}` | Consulta detalhe | `comunicado:read` |
| POST | `/api/v1/comunicados` | Cria rascunho | `comunicado:create` |
| PUT | `/api/v1/comunicados/{id}` | Edita comunicado | `comunicado:update` |
| POST | `/api/v1/comunicados/{id}/publicar` | Publica | `comunicado:manage` |
| POST | `/api/v1/comunicados/{id}/arquivar` | Arquiva | `comunicado:manage` |
| POST | `/api/v1/comunicados/{id}/marcar-lido` | Registra leitura do usuario atual | `comunicado:read` |
| GET | `/api/v1/comunicados/{id}/contagem-leituras` | Consulta contagem | `comunicado:read` |
| GET | `/api/v1/comunicados/{id}/leituras` | Consulta lista administrativa de leituras | `comunicado:manage` |

Observacao importante: o nome da classe e legado (`NoticiasController`), mas a rota e o agregado novo de `Comunicado`. Nao renomear por estetica sem verificar impacto em DI, testes e compatibilidade do frontend.

### `/api/v1/notificacoes`

Controller: `Simpleto.Api/Controllers/v1/NotificacoesController.cs`.

| Metodo | Rota | Funcao |
|---|---|---|
| POST | `/api/v1/notificacoes` | Registra uma notificacao |
| GET | `/api/v1/notificacoes/minhas` | Lista notificacoes do usuario atual |
| PUT | `/api/v1/notificacoes/canais/{id}/status` | Atualiza status reportado pelo adapter |
| POST | `/api/v1/notificacoes/canais/{id}/reenviar` | Reenvia um canal, preservando o original |
| PUT | `/api/v1/notificacoes/canais/{id}/lida` | Marca canal como lido pelo destinatario |
| GET | `/api/v1/notificacoes/origem/{tipo}/{id}` | Consulta notificacoes de uma origem |
| GET | `/api/v1/notificacoes/condominios/{id}/canais` | Lista canais do condominio |
| PUT | `/api/v1/notificacoes/condominios/{id}/canais` | Habilita/desabilita canal |
| GET | `/api/v1/notificacoes/minhas/preferencias` | Lista preferencias do usuario |
| PUT | `/api/v1/notificacoes/minhas/preferencias` | Atualiza preferencias do usuario |

A migration V071 declara expressamente que a fatia N1 registra o motor/log e nao executa disparo externo. Portanto, nao marcar e-mail ou WhatsApp como entregues sem adapter, worker, provedor e teste de integracao reais.

## Regras que nao podem ser perdidas

1. O tenant deve vir do contexto autenticado; nao confiar em `TenantId` enviado pelo cliente.
2. Toda consulta e mutacao precisa respeitar o tenant e o escopo de autorizacao.
3. Leitura de comunicado e acompanhamento; nao e ciencia formal.
4. Comunicado nao substitui convocacao formal de assembleia.
5. Falha de um canal de notificacao nao deve apagar a notificacao nem interromper o evento de origem.
6. Reenvio deve preservar o registro original e criar rastreabilidade do novo envio.
7. `NotificacaoFinanceira` e um conceito existente separado; nao misturar com a notificacao generica sem decisao documentada.
8. Nao criar uma nova permissao financeira sem ler a decisao registrada em `REQ-20260723-comunicados-notificacoes-modelo-conceitual.md`.

## Pendencias para fechar P1

### P1.1 - Validacao de compilacao e testes

- [ ] Executar `dotnet restore` na solution backend.
- [ ] Executar `dotnet build` na solution backend.
- [ ] Executar `dotnet test` no projeto `Simpleto.Api.Tests`.
- [ ] Registrar os comandos, quantidade de testes e resultado neste documento.
- [ ] Investigar falhas sem corrigir problemas fora deste dominio.

### P1.2 - Validacao de banco

- [ ] Identificar o ambiente alvo (local, homologacao ou Supabase).
- [ ] Confirmar aplicacao de V071, V072, V073 e V074.
- [ ] Confirmar RLS habilitado e `app.tenant_id` preenchido pela aplicacao.
- [ ] Confirmar existencia das funcoes/tabelas de auditoria.
- [ ] Testar rollback apenas em ambiente descartavel; nao executar o bloco DOWN comentado em producao.

### P1.3 - Validacao de autorizacao e tenant

- [ ] Usuario sem autenticacao recebe 401.
- [ ] Usuario sem permissao de leitura recebe 403.
- [ ] Usuario sem permissao de criacao/gestao nao consegue publicar ou arquivar.
- [ ] Usuario de tenant A nao consulta nem altera dados do tenant B.
- [ ] Morador nao consulta leituras de outros usuarios.
- [ ] Perfil administrativo consulta apenas o escopo autorizado.

### P1.4 - Validacao funcional da API

- [ ] Criar rascunho com payload valido.
- [ ] Rejeitar payload sem titulo, resumo, canal ou segmentacao valida conforme regra do handler.
- [ ] Editar rascunho.
- [ ] Publicar rascunho.
- [ ] Impedir transicao invalida de status.
- [ ] Arquivar publicado.
- [ ] Listar por canal, periodo e leitura.
- [ ] Marcar leitura sem duplicar registro.
- [ ] Registrar notificacao com um ou mais canais.
- [ ] Atualizar falha de canal com motivo.
- [ ] Reenviar preservando o registro original.
- [ ] Marcar canal portal como lido pelo destinatario correto.

### P1.5 - Validacao frontend

- [ ] Encontrar servicos Angular que chamam `/api/v1/comunicados`.
- [ ] Encontrar servicos Angular que chamam `/api/v1/notificacoes`.
- [ ] Confirmar se ha telas administrativas reais ou apenas stubs.
- [ ] Confirmar tratamento de 401, 403, 404 e erro de validacao.
- [ ] Confirmar envio do contexto de tenant pelo interceptor existente.
- [ ] Registrar telas ausentes no documento P2 ou P3, sem marcar P1 como falha de backend.

## Criterio de pronto da P1

P1 pode ser marcada como concluida somente quando:

- build e testes do backend passam;
- migrations necessarias estao aplicadas no ambiente alvo;
- os cenarios de tenant e permissao passam;
- os fluxos basicos de comunicado e notificacao foram exercitados;
- o frontend esta integrado ou as telas ausentes foram explicitamente transferidas para P2/P3;
- este documento e o checklist principal contem a evidencia da validacao.

## Questoes que exigem decisao humana

- Qual e o ambiente alvo para a primeira validacao real?
- O campo de segmentacao deve aceitar comunicado para todo o tenant sem registro em `comunicadosegmento`, ou essa regra ja esta fixada no handler?
- Qual e a politica de retencao para leitura e auditoria?
- Quais eventos de dominio geram notificacoes na primeira entrega?
- O portal e canal obrigatorio e nao pode ser desabilitado para todos os usuarios?
- Quem administra credenciais, templates e custos do WhatsApp?
