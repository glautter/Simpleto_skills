# Checklist de implementacao e continuidade

Data de atualizacao: 2026-09-27

> Este arquivo e o indice de continuidade. O detalhamento tecnico e funcional esta em [docs/implementacao-comunicados-notificacoes/README.md](docs/implementacao-comunicados-notificacoes/README.md). Nao implemente um item apenas pelo nome da caixa: leia o documento vinculado, confirme o estado no codigo e registre a evidencia de validacao.

## Como outro modelo deve continuar

1. Leia este arquivo inteiro.
2. Leia [README.md](docs/implementacao-comunicados-notificacoes/README.md).
3. Leia o documento da prioridade marcada como `EM ANDAMENTO`.
4. Confirme o estado real no codigo indicado em "evidencias".
5. Execute a validacao indicada antes de editar.
6. Faca uma alteracao pequena, rode novamente a validacao e atualize este checklist.
7. Nao marque um item como concluido sem registrar arquivo alterado, comando executado e resultado.

## Status geral

- [x] P0 — Indicador de versão visível no app shell
- [x] P0 — Modal de “Últimas atualizações” com linguagem natural para usuário final
- [x] P0 — Build validado com sucesso no frontend
- [~] P1 — Base do modulo de comunicados/notificacoes (parcialmente implementada)
- [ ] P2 — Gestão administrativa do conteúdo
- [ ] P3 — Experiência do usuário final no portal/morador
- [ ] P4 — Integrações de canal e notificação
- [ ] P5 — Testes, segurança, observabilidade e release

## P0 — Concluído

- [x] Botão/indicador de versão visível no header global
- [x] Clique abre modal/dialog com release notes
- [x] Conteúdo escrito em linguagem simples para o usuário final
- [x] Build do projeto validado com sucesso

## P1 — Em andamento

### Objetivo
Implementar e validar a base funcional do modulo de comunicados/notificacoes, antes de avancar para a experiencia do usuario final.

Detalhamento: [P1-BASE-COMUNICADOS-NOTIFICACOES.md](docs/implementacao-comunicados-notificacoes/P1-BASE-COMUNICADOS-NOTIFICACOES.md)

Estado confirmado no repositorio:
- [x] Fatia C1: agregado, persistencia, API e fluxo basico de Comunicado existem.
- [x] Fatia N1: motor/registro de Notificacao e API administrativa existem.
- [x] Cobertura de frontend administrativo e portal do morador confirmada (telas reais, nao stub — ver P1.5).
- [ ] Disparo real de e-mail/WhatsApp ainda nao deve ser considerado concluido apenas porque o registro de notificacao existe.
- [x] Testes e validacao de ambiente executados para C1/N1 (build/test backend + estrutura de banco no Supabase alvo).

### Checklist
- [x] Definir entidade/estrutura de dados de Comunicado
- [x] Definir entidade/estrutura de dados de Notificacao
- [x] Definir relacao com destinatarios e grupos
- [x] Definir status do fluxo (rascunho, publicado, arquivado; notificacao com status por canal)
- [x] Definir campos obrigatorios e regras de validacao
- [x] Modelar endpoints minimos da API
- [x] Implementar listagem de comunicados
- [x] Implementar criacao de comunicado
- [x] Implementar edicao de comunicado
- [x] Implementar publicacao de comunicado
- [x] Implementar arquivamento de comunicado
- [x] Implementar registro de leitura
- [x] Implementar registro de notificacao e status por canal
- [x] Implementar historico/auditoria estrutural (RLS + audit trigger quando disponivel)
- [x] Validar permissoes por perfil/condominio no desenho da API
- [x] Validar fluxos com dados reais de condominio (comunicado=4, notificacao=7 linhas ja existentes no Supabase alvo)
- [x] Validar build/testes do backend apos a implementacao documentada
- [x] Validar telas frontend contra os endpoints reais (servicos e telas reais confirmados; ver P1.5 no documento P1)

## P2 — Gestao administrativa

Detalhamento inicial: [P2-GESTAO-ADMINISTRATIVA.md](docs/implementacao-comunicados-notificacoes/P2-GESTAO-ADMINISTRATIVA.md)

- [ ] Tela de listagem de comunicados
- [ ] Filtros por categoria, status e período
- [ ] Criação/edição de comunicado
- [ ] Publicação programada
- [ ] Visualização de histórico e alterações
- [ ] Ações de arquivar, cancelar e reabrir

## P3 — Experiencia do usuario final

Detalhamento: [P3-EXPERIENCIA-USUARIO-FINAL.md](docs/implementacao-comunicados-notificacoes/P3-EXPERIENCIA-USUARIO-FINAL.md)

- [ ] Listagem para moradores/usuários
- [ ] Indicador de comunicado novo
- [ ] Visualização detalhada
- [ ] Leitura/confirmar visualização, se houver regra
- [ ] Organização por prioridade ou categoria
- [ ] Confirmar qual frontend/portal hospeda a experiencia do morador
- [ ] Validar estados vazio, carregando, erro, lido e nao lido
- [ ] Validar isolamento de tenant e escopo do usuario final
- [ ] Validar responsividade e acessibilidade

## P4 — Integracoes

Detalhamento: [P4-INTEGRACOES-CANAIS.md](docs/implementacao-comunicados-notificacoes/P4-INTEGRACOES-CANAIS.md)

- [ ] E-mail
- [ ] Push/alerta interno
- [ ] WhatsApp/SMS, se necessário
- [ ] Política de retry e fallback
- [ ] Log de envio
- [ ] Confirmar provedores e credenciais por ambiente
- [ ] Implementar adapters isolados do dominio
- [ ] Implementar worker/fila e idempotencia
- [ ] Testar falhas transitorias, definitivas e callbacks duplicados

## P5 — Qualidade e producao

Detalhamento: [P5-QUALIDADE-SEGURANCA-OPERACAO.md](docs/implementacao-comunicados-notificacoes/P5-QUALIDADE-SEGURANCA-OPERACAO.md)

- [ ] Testes unitários
- [ ] Testes de integração
- [ ] Cenários de regressão
- [ ] Validação de segurança e permissões
- [ ] Monitoramento e logs
- [ ] Rollback e documentação de operação
- [ ] Testar isolamento cross-tenant e autorizacao
- [ ] Configurar metricas e alertas acionaveis
- [ ] Validar pipeline de homologacao e smoke test
- [ ] Ensaiar rollback de aplicacao e migration

## Evidencias e decisoes

- Documento conceitual: [modelo conceitual](Simpleto.BackApi/docs/requirements/REQ-20260723-comunicados-notificacoes-modelo-conceitual.md)
- Requisitos e historias: [epico e historias](Simpleto.BackApi/docs/requirements/REQ-20260723-comunicados-notificacoes-epico-historias.md)
- Migration principal: [V071](Simpleto.BackApi/Simpleto.Infrastructure/Migrations/V071__comunicado_notificacao_tables.sql)
- Migration da regra de convocacao: [V072](Simpleto.BackApi/Simpleto.Infrastructure/Migrations/V072__comunicado_convocacao_assembleia.sql)
- Controller de comunicados: [NoticiasController.cs](Simpleto.BackApi/Simpleto.Api/Controllers/v1/NoticiasController.cs)
- Controller de notificacoes: [NotificacoesController.cs](Simpleto.BackApi/Simpleto.Api/Controllers/v1/NotificacoesController.cs)

## Regra de avanco

- [ ] P0 concluído antes de P1
- [ ] P1 concluído antes de P2
- [ ] P2 concluído antes de P3
- [ ] P3 concluído antes de P4
- [ ] P4 concluído antes de P5

## Proximo passo recomendado

- [x] Fechar a validacao da P1: rodar testes/build do backend, confirmar migration aplicada no ambiente alvo e verificar a integracao das telas frontend com `/api/v1/comunicados` e `/api/v1/notificacoes`. (2026-09-27, evidencia completa em P1.1/P1.2/P1.5 no documento P1)

Pendencias que impedem marcar P1 100% concluida:

- [x] P1.2: bug de `to_regclass` no bloco de auditoria de V071 corrigido via `V121__fix_comunicado_notificacao_audit_trigger_check.sql`, aplicada no Supabase alvo com autorizacao do usuario. 5 triggers de auditoria confirmados (`information_schema.triggers`, 15 linhas = 5 tabelas x 3 eventos).
- [x] P1.4: fluxos funcionais via HTTP validados com usuario de teste real (`admin.hml@simpleto.local`) contra API local apontando pro Supabase alvo — CRUD de comunicado, transicoes de status, filtros, leitura idempotente, registro/reenvio/status de notificacao. Achado: `POST /notificacoes` e `.../reenviar` retornam `Canais[].Id`/`DestinatarioId` zerados (bug de mapeamento, dado no banco esta correto) — registrado em P1.4, nao corrigido ainda.
- [~] P1.3 (parcial): 401 sem token e escopo por tenant confirmados. 403 (permissao insuficiente) e isolamento cross-tenant **nao testados** — so havia uma credencial `TenantMasterAdmin` de um unico tenant.
- Novo achado (ja incorporado): a premissa "P2/P3 nao iniciadas" estava errada — ja existem telas reais de gestao (P2) e leitura no portal do morador (P3). Essas secoes precisam ser revisadas item a item antes de assumir o que falta.
- Novo achado 2026-09-27: bug de mapeamento no DTO de resposta de `RegistrarNotificacaoHandler`/`ReenviarNotificacaoCanalHandler` (ver P1.4) — nao bloqueia P1, mas precisa ser resolvido antes de P4.

## Observação

Este arquivo serve como base de continuidade para o projeto, em vez de depender apenas da memoria de sessao. Sempre que uma etapa for concluida, marque a caixa correspondente e mantenha o status atualizado. O simbolo `[~]` significa "em andamento/parcial", nao concluido.
