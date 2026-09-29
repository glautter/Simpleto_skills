# P5 - Qualidade, seguranca, observabilidade e release

Data de atualizacao: 2026-09-29
Status: EM ANDAMENTO — gap de RBAC/plano corrigido; CI/CD, observabilidade e massa de teste completa seguem pendentes (dependem de decisao de infraestrutura/operacao, ver "Questoes em aberto")

## Objetivo

Garantir que comunicados e notificacoes possam ser liberados sem regressao silenciosa, vazamento entre tenants, perda de historico ou falha de operacao sem alerta.

P5 e uma porta de release. Nao significa que todos os testes do sistema inteiro precisam ser reescritos, mas todo risco introduzido por este dominio precisa ter evidencia proporcional.

## Piramide de testes

### Unitarios

Cobrir regras isoladas:

- transicoes de status de comunicado;
- validacao de payload;
- validacao de segmentacao;
- proibicao de convocacao formal por comunicado;
- deduplicacao de leitura;
- deduplicacao de notificacao por canal;
- classificacao de erro transitorio/definitivo;
- reenvio preservando registro original.

### Integracao backend

Cobrir:

- repository e SQL contra banco de teste;
- constraints de status/canal;
- unicidade de leitura;
- unicidade de canal por notificacao;
- RLS e `tenantid`;
- auditoria de insert/update/delete;
- comandos e queries com permissao correta.

### API

Para cada endpoint relevante, verificar:

- 200/201/204 no sucesso conforme contrato;
- 400 para payload invalido;
- 401 sem autenticacao;
- 403 sem permissao;
- 404 para id inexistente ou fora do escopo;
- tenant A nao acessa tenant B;
- resposta nao expõe dados de outros moradores.

### Frontend/E2E

Cobrir os fluxos criticos:

- criar e publicar comunicado;
- morador lista, abre e marca como lido;
- administrador consulta leitura;
- notificacao com sucesso/falha visivel;
- sessao expirada e erro de permissao.

E2E deve ser reservado a fluxos completos; regra de negocio simples deve permanecer em teste unitario/integracao.

## Massa minima

- dois tenants distintos;
- dois condominios ou escopos dentro do tenant;
- usuario administrativo com permissao;
- usuario sem permissao;
- morador do escopo correto;
- morador fora do escopo;
- comunicado rascunho, publicado e arquivado;
- notificacao com canal portal, e-mail e WhatsApp;
- canal habilitado, desabilitado, com opt-out e com falha;
- leitura existente e tentativa de duplicidade.

Nao usar dados pessoais reais em teste, homologacao ou logs.

## Seguranca (revisado 2026-09-29)

- [x] Tenant vem do contexto autenticado, nao do payload confiado — confirmado via `ITenantContext.TenantId`/`IUserContext.UserId` em todos os handlers/controllers de notificacao (`NotificacoesController.cs`); nenhum comando le tenant/usuario do payload.
- [x] Policies RBAC sao aplicadas na API e verificadas nos handlers — **achado e corrigido 2026-09-29**: `NotificacoesController` nao tinha nenhuma policy por endpoint (so `[Authorize]` generico), diferente de `NoticiasController` (`comunicado:*`). Corrigido: migration `V122__seed_notificacao_rbac_e_plano.sql` cria as permissoes `notificacao:{create,read,update,delete,manage}`, espelha 1:1 os grants por role ja existentes para `comunicado` (21 grants, 8 roles), e adiciona `notificacao` aos 4 planos que ja tem `comunicado` (Legado, Basico, Profissional, Enterprise) — o sistema tem um gate de plano (`PermissionRequirementHandler`/`plano_recurso`) que roda antes do RBAC e bloquearia tudo sem isso. Endpoints administrativos/sistema (`Post` registrar, `AtualizarStatusCanal`, `DefinirCanalCondominio`) ganharam `notificacao:create`/`notificacao:manage`; endpoints self-service com ownership check no handler (`GetMinhas`, `MarcarComoLida`, `ReenviarCanal`, preferencias) ganharam `notificacao:read` como piso (todas as roles tem pelo menos read). Aplicada no Supabase alvo, validada via HTTP real (login `admin.hml@simpleto.local`): `GET minhas` 200, `GET minhas/preferencias` 200, `POST notificacoes` 200, `PUT canais/{id}/lida` 204, sem token 401, token invalido 401. Build + `dotnet test --filter FullyQualifiedName~Notificacoes` (36/36) sem regressao.
- [x] RLS esta habilitado — confirmado em `V071__comunicado_notificacao_tables.sql` (bloco `FOREACH v_table IN ARRAY ['comunicado', 'comunicadosegmento', 'leituracomunicado', 'notificacao', 'notificacaocanal']`, `ENABLE ROW LEVEL SECURITY` + `CREATE POLICY tenant_isolation_*`). **Nao testado** com dado cross-tenant real no banco alvo (mesma ressalva do P1.3 — falta segunda credencial/tenant).
- [ ] Escopo por condominio/bloco/unidade/perfil nao permite ampliacao pelo cliente — nao verificado em profundidade nesta revisao (segmentacao de comunicado nao foi reauditada).
- [x] Morador nao ve leitura de outro morador — `GetMinhasNotificacoesHandler` usa `_userContext.UserId.Value`, nao aceita id de outro usuario via payload/query.
- [x] Conteudo nao permite injecao de HTML/script no portal — revisado: nenhum uso de `innerHTML`/`bypassSecurityTrust` no modulo de comunicacao do frontend (`grep` sem resultado); Angular interpola com escape automatico por padrao.
- [ ] Webhooks de provedores validam autenticidade e idempotencia — nao aplicavel na integracao atual (SMTP e OpenClaw sao chamadas sincronas de saida, sem webhook de retorno confirmado); a confirmar se OpenClaw usa callback assincrono em algum fluxo.
- [~] Tokens, senhas, numeros completos e conteudo sensivel nao aparecem nos logs — `EmailDispatchService`/`WhatsAppDispatchService` logam `ex.Message` em falha (pode conter fragmento do provedor, nao senha/token); nao auditado a fundo.
- [ ] Segredos ficam fora do repositorio e sao rotacionaveis — **violado, nao resolvido**: `Simpleto.Api/appsettings.json` tem senha de app do Gmail em texto puro, versionada no repo desde antes desta sessao. Recomendacao mantida: revogar a senha atual no Google e mover para user-secrets/variavel de ambiente.

Para comunicacoes financeiras ou com efeito juridico, encaminhar a validacao para a analise juridica especifica; log de envio nao substitui automaticamente prova formal.

## Observabilidade

### Logs estruturados

Cada operacao deve permitir localizar:

- request/correlation id;
- tenant id, quando seguro e necessario;
- entidade e id da origem;
- notificacao/canal;
- tentativa;
- resultado e motivo;
- duracao.

Nao registrar corpo completo de comunicacao ou documento sensivel por padrao.

### Metricas

Minimo recomendado:

- quantidade de comunicados criados/publicados/arquivados;
- tempo de listagem e publicacao;
- notificacoes por canal e status;
- taxa de falha por provedor/canal;
- idade da fila pendente;
- quantidade de retries e falhas definitivas;
- erros 401/403/404/5xx;
- operacoes por tenant para detectar anomalia.

### Alertas

Cada alerta precisa indicar responsavel e acao:

- fila pendente acima do limite -> operador verifica worker/provedor;
- aumento de falhas por canal -> verificar credencial, contrato ou indisponibilidade;
- falhas de RLS/autorizacao -> bloquear promocao e investigar vazamento;
- migration incompleta -> impedir release;
- erro 5xx acima do limite -> rollback ou mitigacao definida.

Os limites numericos devem ser definidos com a infraestrutura real; nao inventar limiares no codigo sem acordo operacional.

## CI/CD e ambientes

Ambientes minimos:

- desenvolvimento: dados descartaveis e provedores sandbox/mock;
- homologacao: topologia proxima da producao e envio controlado;
- producao: segredos reais, dados reais e aprovacao de release.

Pipeline recomendado:

1. restore/dependencias;
2. build;
3. testes unitarios;
4. testes de integracao/API;
5. analise estatica e vulnerabilidades;
6. empacotamento versionado;
7. migration controlada;
8. deploy em homologacao;
9. smoke test;
10. aprovacao/promocao para producao.

A imagem/artefato testado em homologacao deve ser o mesmo promovido para producao. Segredos nao entram no repositorio nem no output do pipeline.

## Migration e rollback

- Migration de banco deve ser aplicada antes do codigo que depende dela quando houver compatibilidade; se nao houver, fazer deploy em duas fases.
- Confirmar backup/ponto de restauracao antes de migration de producao.
- Preferir migrations aditivas e reversiveis; nao apagar historico de comunicacao.
- O rollback da aplicacao nao desfaz automaticamente uma migration destrutiva.
- Registrar como a versao anterior se comporta com o schema novo.
- Definir quem autoriza rollback e como os envios pendentes sao tratados.

## Criterio de pronto da P5

- [ ] Suites definidas para backend, API e frontend passam.
- [ ] Cenarios negativos e cross-tenant passam.
- [ ] Vulnerabilidades e segredos foram verificados.
- [ ] Logs, metricas e alertas estao acessiveis para a equipe responsavel.
- [ ] Deploy em homologacao foi executado com smoke test.
- [ ] Plano de rollback foi testado ou ensaiado.
- [ ] Checklist de release e evidencias foram atualizados.

## Questoes em aberto

- Qual ferramenta executa CI/CD e onde ficam os artefatos?
- Qual e a estrategia real de deploy e rollback do projeto?
- Qual banco/ambiente sera usado para os testes integrados?
- Quem recebe alertas de fila, falha de provedor e erro de autorizacao?
- Qual politica de retencao de logs, auditoria, leitura e notificacao?
