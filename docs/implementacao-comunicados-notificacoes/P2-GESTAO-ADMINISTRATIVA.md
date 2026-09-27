# P2 - Gestao administrativa de Comunicados

Data de atualizacao: 2026-09-27
Status: NAO INICIADA COMO ENTREGA

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

- [ ] Com permissao de criacao, o usuario consegue salvar um rascunho valido.
- [ ] Campos invalidos impedem o envio e mostram o erro no campo correspondente.
- [ ] Ao editar, o id da rota nao pode ser substituido por id informado em outro campo.
- [ ] Sem permissao, a API rejeita a operacao e a tela nao informa sucesso falso.
- [ ] Apos salvar, a tela mostra o status real retornado pela API.

### Publicar e arquivar

- [ ] Publicar exige permissao de gestao.
- [ ] A tela pede confirmacao antes da publicacao.
- [ ] Apos publicar, o botao de editar respeita a regra real do backend.
- [ ] Arquivar exige confirmacao e atualiza a lista sem apagar o historico.
- [ ] O aviso de convocacao aparece quando aplicavel.

### Filtros e leitura

- [ ] Os filtros sao enviados nos nomes exatos aceitos pela API.
- [ ] A lista nao exibe comunicados fora do tenant ou do escopo do usuario.
- [ ] Marcar como lido atualiza a linha/detalhe sem criar duplicidade.
- [ ] A consulta de leituras de terceiros so aparece para quem possui gestao.

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

- [ ] Lista, filtros, formulario, publicacao e arquivamento funcionam contra a API real.
- [ ] Estados de erro e permissao foram testados.
- [ ] Segmentacao foi exercitada com pelo menos dois escopos.
- [ ] A tela funciona em desktop e viewport menor sem cortar campos ou acoes.
- [ ] Testes frontend relevantes passam.
- [ ] Checklist principal registra arquivos alterados e comando de validacao.

## Proximo documento a criar depois da P2

Quando P2 estiver validada, criar `P3-EXPERIENCIA-USUARIO-FINAL.md` com a jornada do morador, sino de notificacoes, detalhe do comunicado, leitura e acessibilidade. Nao antecipar esse documento como implementacao sem confirmar se o portal/app do morador esta neste mesmo workspace.
