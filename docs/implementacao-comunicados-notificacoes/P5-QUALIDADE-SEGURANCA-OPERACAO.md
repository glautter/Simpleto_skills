# P5 - Qualidade, seguranca, observabilidade e release

Data de atualizacao: 2026-09-27
Status: NAO CONCLUIDA

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

## Seguranca

- [ ] Tenant vem do contexto autenticado, nao do payload confiado.
- [ ] Policies RBAC sao aplicadas na API e verificadas nos handlers.
- [ ] RLS esta habilitado e testado no banco alvo.
- [ ] Escopo por condominio/bloco/unidade/perfil nao permite ampliacao pelo cliente.
- [ ] Morador nao ve leitura de outro morador.
- [ ] Conteudo nao permite injecao de HTML/script no portal.
- [ ] Webhooks de provedores validam autenticidade e idempotencia.
- [ ] Tokens, senhas, numeros completos e conteudo sensivel nao aparecem nos logs.
- [ ] Segredos ficam fora do repositorio e sao rotacionaveis.

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
