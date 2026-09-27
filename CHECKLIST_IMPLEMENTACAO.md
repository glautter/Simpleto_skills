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
- [x] P1 — Base do modulo de comunicados/notificacoes (concluida, com ressalva: 403/isolamento cross-tenant nao testados por falta de segunda credencial — decisao do usuario foi avancar mesmo assim)
- [~] P2 — Gestão administrativa do conteúdo (parcial — edição, confirmação e filtro server-side de canal/período implementados 2026-09-27; falta publicação programada (exige mudança de API) e teste no navegador — ver P2-GESTAO-ADMINISTRATIVA.md)
- [~] P3 — Experiência do usuário final no portal/morador (parcial; achado de segurança em reenviar corrigido — ver P3-EXPERIENCIA-USUARIO-FINAL.md)
- [~] P4 — Integrações de canal e notificação (achado 2026-09-27: e-mail e WhatsApp JA implementados e rodando via workers, de commits anteriores a esta sessão — item estava marcado errado como "não iniciado". Corrigido 2026-09-27: claim atômico com `FOR UPDATE SKIP LOCKED` para evitar envio duplicado multi-instância. Gaps restantes: segredo SMTP versionado, sem push/SMS, sem teste de callback duplicado real — ver P4-INTEGRACOES-CANAIS.md)
- [ ] P5 — Testes, segurança, observabilidade e release

## P0 — Concluído

- [x] Botão/indicador de versão visível no header global
- [x] Clique abre modal/dialog com release notes
- [x] Conteúdo escrito em linguagem simples para o usuário final
- [x] Build do projeto validado com sucesso

## P1 — Concluida (com ressalva)

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

## P2 — Gestao administrativa (parcial — revisao de codigo + correcoes em 2026-09-27, sem teste manual no navegador)

Detalhamento: [P2-GESTAO-ADMINISTRATIVA.md](docs/implementacao-comunicados-notificacoes/P2-GESTAO-ADMINISTRATIVA.md)

- [x] Tela de listagem de comunicados (`ComunicadosGestaoComponent`, rota `comunicados-gestao`)
- [x] Filtros por categoria, status e período — canal/período agora são server-side (query params reais na API); status continua client-side porque a API não aceita esse filtro (limitação de API, documentada)
- [x] Criação/edição de comunicado — edição implementada 2026-09-27 (botão "Editar" no detalhe, reaproveitando o formulário; `ComunicadosService.update()` agora é usado)
- [ ] Publicação programada — nao existe (nem na API)
- [~] Visualização de histórico e alterações — lista de leituras existe; log de alteracoes/auditoria nao aparece na UI
- [x] Ações de arquivar (com confirmação, implementada 2026-09-27); cancelar/reabrir não existem — nao correspondem ao ciclo de vida real da API (`rascunho -> publicado -> arquivado`)
- Validado com `npm run build` (Angular production) — sucesso, sem erros novos. `ng test` e teste manual no navegador ainda pendentes.

## P3 — Experiencia do usuario final

Detalhamento: [P3-EXPERIENCIA-USUARIO-FINAL.md](docs/implementacao-comunicados-notificacoes/P3-EXPERIENCIA-USUARIO-FINAL.md)

- [x] Listagem para moradores/usuários (`ComunicadosLeituraComponent`, reaproveitado nos portais condominio/morador)
- [x] Indicador de comunicado novo (coluna "Leitura" + cards de contagem nao-lidos/lidos)
- [x] Visualização detalhada — implementada 2026-09-27 (`ComunicadoLeituraDetalheComponent`, rota `comunicados/:id` nos dois portais), não testada manualmente no navegador
- [x] Leitura/confirmar visualização, se houver regra (`marcarComoLido`, com dialog de confirmação, validado em P1.4)
- [ ] Organização por prioridade ou categoria — nao ha campo de prioridade na entidade
- [x] Confirmar qual frontend/portal hospeda a experiencia do morador — resolvido: mesmo app, portal `morador`
- [~] Validar estados vazio, carregando, erro, lido e nao lido — parcial, so revisão de código
- [~] Validar isolamento de tenant e escopo do usuario final — tenant ok; **achado de segurança corrigido**: reenviar agora checa posse (`DestinatarioId == UserId`, teste unitário cobrindo 403); `atualizar-status` segue sem checagem, decisão adiada para P4 (ver P3, "Achado de segurança")
- [ ] Validar responsividade e acessibilidade — não verificado no navegador

## P4 — Integracoes (parcial — e-mail e WhatsApp ja implementados, revisado 2026-09-27)

Detalhamento: [P4-INTEGRACOES-CANAIS.md](docs/implementacao-comunicados-notificacoes/P4-INTEGRACOES-CANAIS.md)

- [x] E-mail — `EmailDispatchService` + `EmailDispatcherWorker`, via SMTP (`IEmailSenderService`)
- [ ] Push/alerta interno — fora de escopo confirmado (so Portal, E-mail, WhatsApp)
- [x] WhatsApp — `WhatsAppDispatchService` + `WhatsAppDispatcherWorker`, via OpenClaw (`IWhatsAppSender`); SMS fora de escopo
- [x] Política de retry e fallback — 3 tentativas, backoff de 5min, status `falhou` apos esgotar
- [x] Log de envio — `FalhaMotivo`/`Tentativas`/`UltimaTentativaEm` na tabela `notificacaocanal`
- [x] Confirmar provedores e credenciais por ambiente — SMTP e OpenClaw ja configurados (mas ver gap de segredo abaixo)
- [x] Implementar adapters isolados do dominio — `IEmailSenderService`, `IWhatsAppSender`
- [x] Implementar worker/fila e idempotencia — worker roda (polling 30s sobre a tabela); idempotencia entre multiplas instancias corrigida 2026-09-27 com claim atomico `FOR UPDATE SKIP LOCKED`
- [ ] Testar falhas transitorias, definitivas e callbacks duplicados — so testado com mocks unitarios (26/26 passando); sem teste de reentrega real/callback duplicado
- [ ] **Gap de seguranca nao corrigido**: `Simpleto.Api/appsettings.json` tem senha SMTP em texto puro versionada no repo — viola "credenciais fora do repositorio"

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

- [x] P0 concluído antes de P1
- [x] P1 concluído antes de P2 (com ressalva de P1.3 documentada)
- [ ] P2 concluído antes de P3
- [ ] P3 concluído antes de P4
- [ ] P4 concluído antes de P5

## Proximo passo recomendado

- [x] Fechar a validacao da P1: rodar testes/build do backend, confirmar migration aplicada no ambiente alvo e verificar a integracao das telas frontend com `/api/v1/comunicados` e `/api/v1/notificacoes`. (2026-09-27, evidencia completa em P1.1/P1.2/P1.5 no documento P1)
- [ ] Avancar para P2 (Gestao administrativa): revisar item a item o que ja existe no frontend (`comunicados-gestao`, `comunicado-detalhe`, `notificacoes-config` — ver P1.5) contra o escopo de [P2-GESTAO-ADMINISTRATIVA.md](docs/implementacao-comunicados-notificacoes/P2-GESTAO-ADMINISTRATIVA.md), em vez de assumir que nada foi feito.

P1 fechada em 2026-09-27. Resumo do que foi corrigido/validado nesta sessao:

- [x] P1.1: build/testes do backend (636/636 aprovados).
- [x] P1.2: bug de `to_regclass` no bloco de auditoria de V071 corrigido via `V121__fix_comunicado_notificacao_audit_trigger_check.sql`, aplicada no Supabase alvo. 5 triggers de auditoria confirmados (`information_schema.triggers`, 15 linhas = 5 tabelas x 3 eventos).
- [x] P1.4: fluxos funcionais via HTTP validados com usuario de teste real (`admin.hml@simpleto.local`) contra API local apontando pro Supabase alvo — CRUD de comunicado, transicoes de status, filtros, leitura idempotente, registro/reenvio/status de notificacao.
- [x] Bug de mapeamento no DTO de resposta de `RegistrarNotificacaoHandler`/`ReenviarNotificacaoCanalHandler` (`Canais[].Id`/`DestinatarioId` zerados) corrigido, testado (636/636) e validado via HTTP.
- [x] P1.5: telas reais de gestao/leitura confirmadas no frontend (nao stub) — corrige a premissa antiga de que P2/P3 "nao tinham nada".
- [~] P1.3 (ressalva aceita, nao bloqueante): 401 sem token e escopo por tenant confirmados. 403 (permissao insuficiente) e isolamento cross-tenant **nao testados** — so havia uma credencial `TenantMasterAdmin` de um unico tenant. Decisao do usuario: seguir para P2 mesmo assim; revisitar quando houver uma segunda credencial/tenant de teste.

## Observação

Este arquivo serve como base de continuidade para o projeto, em vez de depender apenas da memoria de sessao. Sempre que uma etapa for concluida, marque a caixa correspondente e mantenha o status atualizado. O simbolo `[~]` significa "em andamento/parcial", nao concluido.
