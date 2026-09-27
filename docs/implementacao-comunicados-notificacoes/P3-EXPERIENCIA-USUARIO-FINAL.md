# P3 - Experiencia do usuario final

Data de atualizacao: 2026-09-27
Status: NAO INICIADA COMO ENTREGA

## Objetivo

Permitir que o morador ou usuario final encontre, entenda e acompanhe comunicados e notificacoes no canal destinado a ele, com linguagem clara, poucos passos e sem confundir leitura com ciencia formal.

## Decisao de produto necessaria antes de codar

O repositorio confirmado nesta etapa e o frontend administrativo Angular. Ainda precisa ser confirmado se a experiencia do morador sera:

- uma area no mesmo frontend;
- um aplicativo separado;
- outro frontend existente no workspace;
- ou somente portal web responsivo.

Nao criar componentes de morador no frontend administrativo por suposicao. Registrar a decisao e o caminho do projeto antes de iniciar a implementacao.

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

## Notificacoes

A experiencia deve diferenciar:

- comunicado: conteudo editorial que pode ser lido no portal;
- notificacao: aviso originado por evento e entregue por canal;
- documento financeiro: documento individualizado, que nao deve aparecer como comunicado generico sem permissao e regra propria.

O sino/central de notificacoes deve informar canal, data, status e origem de forma compreensivel. Nao exibir como entregue algo que a API marcou como falho.

## Criterios de aceite

### Lista

- [ ] Usuario autenticado ve apenas itens do seu tenant e escopo.
- [ ] Usuario ve diferenca clara entre lido e nao lido.
- [ ] Lista vazia tem mensagem orientativa, nao uma tela quebrada.
- [ ] Erro de rede permite tentar novamente.
- [ ] Sessao expirada encaminha para autenticacao sem perder informacao sensivel na URL.
- [ ] Paginacao/rolagem nao duplica itens nem perde filtros.

### Leitura

- [ ] Abrir detalhe registra leitura para o usuario correto.
- [ ] Reabrir nao cria duplicidade.
- [ ] A tela nao afirma que leitura equivale a ciencia formal.
- [ ] Um morador nao consegue consultar a leitura de outro morador.

### Notificacao

- [ ] Portal mostra somente notificacoes destinadas ao usuario.
- [ ] Status de falha e pendencia e compreensivel.
- [ ] Marcar como lida altera apenas o registro do usuario autenticado.
- [ ] Reenvio e acao administrativa, nao uma acao disponivel para qualquer morador.

### Acessibilidade e clareza

- [ ] Fluxo funciona por teclado e leitor de tela conforme o Design System existente.
- [ ] Textos evitam jargao tecnico e juridico desnecessario.
- [ ] Acoes importantes tem nome claro e confirmacao quando houver efeito de estado.
- [ ] Conteudo e controles cabem em viewport mobile sem sobreposicao.

## Dependencias

- P1 validada e contrato de API confirmado.
- Definicao do frontend/portal alvo.
- Modelos e estados de erro documentados.
- P2 ou endpoints administrativos separados do fluxo do morador.
- Revisao visual pela especialista de UI depois da definicao do fluxo.

## Criterio de pronto da P3

- [ ] Jornada de lista, detalhe e leitura funciona no canal escolhido.
- [ ] Isolamento de tenant e autorizacao foram testados.
- [ ] Estados vazio, carregando, erro, nao lido, lido e falha foram validados.
- [ ] Responsividade e acessibilidade foram verificadas em desktop e mobile.
- [ ] Testes frontend e pelo menos um fluxo integrado passam.
- [ ] Checklist registra o frontend correto, arquivos alterados e comando de validacao.

## Questoes em aberto

- Qual projeto/frontend hospeda a experiencia do morador?
- Existe prioridade entre comunicados e notificacoes no primeiro acesso?
- O usuario pode arquivar ou apenas marcar como lido?
- Existe prioridade formal ou apenas data/categoria?
- O historico tem politica de retencao diferente da comunicacao ativa?
