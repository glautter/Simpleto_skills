# Análise do Módulo Financeiro — o que existe, o que falta e plano

## Contexto
Foi pedida uma análise de 9 pontos do fluxo financeiro de uma administradora de condomínios (Contas a Receber, Contas a Pagar/Paybox, Análise Tributária, Portais, Conciliação, Lançamento Projetado, Previsão Orçamentária, Reforma Tributária e NFS-e Nacional), indicando o que o Simpleto já faz e o que não faz, com detalhamento e plano para o que falta.
O código do back-end (`Simpleto.BackApi`) e do front-end (`Simpleto_front_adm`) foi lido para isso, sem nenhuma alteração.


---

## 1. Resumo executivo

| # | Ponto | Status |
|---|---|---|
| 1 | Contas a Receber — acordo | 🟡 Parcial |
| 1 | Contas a Receber — boleto registrado | 🔴 Não feito (gateway falso) |
| 1 | Contas a Receber — envio por e-mail (PDF) / bureau | 🔴 Não feito |
| 2 | Contas a Pagar — núcleo (lançamento, baixa, cancelamento) | 🟢 Feito |
| 2 | Paybox (e-mail/upload + IA/OCR preenche a tela) | 🔴 Não feito |
| 2 | Fluxo operador → acessor/gerente → remessa (com privilégio de pular) | 🟡 Parcial (existe em `Despesa`, desligado de ContaPagar) |
| 2 | Remessa de pagamento ao banco | 🔴 Não feito |
| 3 | Análise tributária antes do CP | 🟡 Parcial (retenção manual, sem motor de regra/calendário) |
| 4 | Portal do Condômino em (quase) tempo real | 🟡 Parcial (lista parcelas; sem sincronização) |
| 4 | Portal do Síndico (CP, CR, balancetes, pastas digitais) | 🟡 Parcial (via portal condomínio; sem pasta digital) |
| 5 | Conciliação bancária (retorno, débito automático) | 🔴 Simulada (tela existe, não lê arquivo) |
| 6 | Lançamento projetado em CP | 🔴 Não feito |
| 7 | Previsão orçamentária por seção do balancete | 🔴 Não feito (falta plano de contas) |
| 8 | Reforma tributária (IBS/CBS) | 🔴 Não feito |
| 9 | Emissão de NFS-e (Padrão Nacional) | 🔴 Não feito |

### ⚠️ Bug crítico encontrado (corrigir antes de tudo)
`Simpleto.Infrastructure/Repositories/ReciboRateioRepository.cs:163` — `BaixarPendentesPorArquivoAsync` (endpoint `POST rateio/recibos/baixar-por-arquivo`) **não lê arquivo nenhum**: executa `UPDATE ... SET status='baixado_arquivo' WHERE tenant_id=@TenantId AND status IN ('gerado','enviado')`. Qualquer clique em "baixar por arquivo de retorno" marca **todas as taxas pendentes da administradora como pagas**. Ação imediata: desabilitar o endpoint/botão (retornar 501) até existir leitura real de retorno CNAB (Fase 2).

### Achados estruturais que afetam vários pontos
- **Dois modelos de recebível que não conversam:** (A) `Cobranca → Parcela` (usado por acordos e portal do condômino) e (B) `RateioMensal → ReciboRateio` (onde vive a taxa condominial). O condômino não vê a taxa condominial.
- **Sem plano de contas:** balancete é lista plana de receitas (`Cobranca`) e despesas (`Despesa` aprovada) — `BalanceteDadosHelper.cs`. Rotas `plano-contas`, `parametros-plano-contas`, `contas-bancarias` no front são `stub`. Bloqueia orçamento, seções do balancete e análise tributária por tipo de serviço.
- **Event bus só em memória** (`ChannelEventBus` + `EventBusProcessor`): eventos se perdem num restart. Remessa, retorno, NFS-e e OCR precisam de fila durável.
- **Storage público** (Supabase): não serve para certificado A1 (.pfx) nem para documentos fiscais sensíveis.
- **E-mail sem anexo:** `IEmailSenderService.EnviarAsync(dest, assunto, html)`.

---

## 2. Análise por ponto

### 2.1 Contas a Receber
**Já existe**
- Acordo: `Acordo.cs`, `AcordosController` (termo PDF, confissão de dívida com hash SHA e verificação, cancelar), `AcordoPdfService`, tela `cobranca-acordos` + `criar-acordo-dialog`.
- Rateio mensal e recibos (`RateioMensalController`, `RecibosRateioController`), parâmetros de emissão com `BancoConvenio`, `LayoutArquivo` (cnab240/400), `NumeroRemessaAtual`.
- Inadimplência, carta de cobrança, relatório para síndico por e-mail.
- Preferência de envio por unidade (`UnidadesFormasEnvio`, `UnidadesEnvioEmailSms`) — só cadastro.

**Não existe**
- Acordo não gera novas parcelas/boletos (só marca a original como `NEGOCIADA`), cobre 1 parcela só e não calcula juros/multa/desconto.
- Boleto real: `IGatewayPagamento` só tem `NoOpGatewayPagamento` (linha digitável falsa). Sem nosso número, código de barras FEBRABAN, registro bancário, CNAB de cobrança.
- "Enviar boletos" (`EnviarBoletosRateioMensalHandler`) só troca status para `enviado` — nenhum e-mail sai. Não há PDF de boleto.
- Modelo bureau (arquivo para gráfica): nada.

**O que precisa ser feito**
1. Unificar recebível: gerar `Parcela` a partir de `ReciboRateio` (ou fazer o recibo ser a parcela), para acordo, portal e conciliação enxergarem a taxa condominial.
2. Integração bancária de cobrança — implementar `IGatewayPagamento` real. Recomendação: começar por **API de boleto híbrido (boleto + Pix QR)** de 1 banco/PSP (ex.: Banco Inter, Itaú, Sicoob ou um agregador tipo Asaas/Celcoin) e, em paralelo, **CNAB 240/400 de cobrança** genérico para bancos sem API.
3. Gerador de PDF de boleto (código de barras I2of5 + QR Pix) reaproveitando o padrão PdfSharpCore de `AcordoPdfService`.
4. `IEmailSenderService` com anexos + worker de envio em lote (reaproveitar `EmailDispatcherWorker`) respeitando a preferência por unidade.
5. Exportação bureau: arquivo de lote (PDF consolidado + CSV/layout da gráfica com endereço de entrega) por unidades marcadas como "impresso".
6. Acordo completo: agrupar N parcelas, cálculo de multa/juros/correção/desconto, gerar parcelas do acordo com boleto, quebra de acordo por inadimplência.

### 2.2 Contas a Pagar, Paybox e fluxo de aprovação
**Já existe**
- `ContaPagar` com fornecedor, conta, NF, bruto/retido/líquido, tipo de pagamento (boleto, TED, Pix, manual, cheque, débito automático), status `programacao → agendado → pago | cancelado`, trava de compensação. CRUD + baixas real (`ContasPagarController`, tela `pagar-principal`).
- Aprovação existe **em outra entidade**: `Despesa` (`PENDENTE_APROVACAO → APROVADA/REJEITADA`, `AprovacaoDespesa`, alçada fixa no código `Valor > 500 ? 2 : 1`).
- Alçada configurável por valor (`AlcadaAprovacaoConfig` + `AlcadaAprovacaoService`) — usada só em Orçamentos.
- RBAC granular `recurso:acao` extensível (`PermissionPoliciesConfiguration.cs` + migration seed).

**Não existe**
- Paybox: nenhum upload na conta a pagar, nenhuma caixa de e-mail de entrada, nenhum cliente de IA/OCR no back-end.
- ContaPagar não passa por aprovação; aprovar uma `Despesa` não gera ContaPagar.
- Papéis operador/acessor/gerente, permissão "lançar sem aprovação", status "fila de remessa".
- Remessa de pagamento (CNAB 240 Pagamentos / Pix / API) e leitura de retorno. "Importar lote" é toast "em evolução".

**O que precisa ser feito**
1. **Novo ciclo de status de ContaPagar:** `rascunho (Paybox) → aguardando_analise_tributaria → aguardando_aprovacao → aprovado (fila de remessa) → em_remessa → pago | rejeitado | cancelado`. Migration converte `programacao` → `aprovado`.
2. **Aprovação na ContaPagar** reaproveitando `AlcadaAprovacaoConfig` (estender para papéis acessor/gerente e para o tipo "conta_pagar") e o padrão de `AprovacaoDespesa`. Aposentar ou converter `Despesa` em ContaPagar (decisão de arquitetura).
3. **Permissão `conta_pagar:lancar_sem_aprovacao`**: se o usuário tem, o lançamento vai direto para `aprovado`/fila de remessa; senão, fila do acessor/gerente. Registro de auditoria em ambos.
4. **Paybox:**
   - Upload (PDF/imagem/XML) e caixa de e-mail dedicada por administradora (IMAP polling ou webhook de provedor de e-mail inbound).
   - Pipeline: XML de NF-e/NFS-e → parse direto (sem IA); boleto → leitura de código de barras/linha digitável (determinístico, valida DV); PDF/imagem livre → LLM com visão (OpenRouter, conforme preferência do projeto) extraindo JSON (fornecedor/CNPJ, valor, vencimento, NF, competência, serviço).
   - Resultado vira ContaPagar `rascunho` com campos pré-preenchidos + score de confiança; operador revisa e confirma. Anexo fica vinculado ao lançamento.
5. **Remessa de pagamento:** gerador CNAB 240 Pagamentos (segmentos J/J-52 boleto, A/B TED/Pix, N/O tributos) por banco + leitura do retorno, ou API bancária de pagamentos. Lote agrupa ContaPagar `aprovado` por conta bancária/data.

### 2.3 Análise tributária (intermediária entre lançamento e CP)
**Já existe** — `RetencaoContaPagar` (ISS, IRRF, INSS, PIS/COFINS/CSLL; percentual manual), `ParametroRecolhimento` (alíquota padrão, código de receita), `GuiaRecolhimento` (pendente/gerada/paga).

**Não existe** — motor de regras (tipo de serviço LC 116, regime do fornecedor/Simples, município do ISS, valor mínimo de retenção, teto INSS), calendário de vencimento por tributo (hoje fixo dia 10), vínculo guia ↔ lançamentos, valor da guia calculado no servidor (hoje confia no front — `GerarGuiaHandler`), DARF/DAM com código de barras.

**O que precisa ser feito**
1. Cadastro tributário do fornecedor (regime, optante Simples, município, CPRB) e do serviço (código LC 116 / item de serviço).
2. Motor de retenção: sugere as retenções ao entrar em `aguardando_analise_tributaria`; analista (permissão `conta_pagar:analise_tributaria`) confirma/ajusta e libera para aprovação.
3. Calendário tributário configurável por tributo (IRRF/PIS-COFINS-CSLL dia 20, INSS dia 20, ISS conforme município, etc.) e geração automática de guias agrupando retenções por tributo/competência — calculado no servidor.
4. Guias entram como ContaPagar de tributo na fila de remessa (CNAB segmento N/O ou código de barras).
5. Preparar para EFD-Reinf (R-2010 / R-4020) — fora do MVP, mas modelar os dados já pensando nele.

### 2.4 Portais (Condômino e Síndico)
**Já existe**
- Condômino: `MeuFinanceiroController` (`GET meu-financeiro/parcelas`), telas boletos-pix, extrato, histórico — só modelo `Parcela`, sem pagar/copiar (gateway falso).
- Síndico: usa o portal "condomínio" por permissão; vê CP, cobrança, inadimplência, balancetes (PDF, pacote assembleia), prestação de contas (lista plana), fechamento de período.

**Não existe**
- Sincronização: nada de SignalR/webhook/job consultando status. Parcela só vira paga por baixa manual.
- Condômino não vê taxa condominial (ReciboRateio), acordos, 2ª via, PDF.
- Portal/visão do síndico read-only dedicada; rota `sindicos` é stub.
- Pasta digital mensal (balancete + extratos + comprovantes + NFs do mês consolidados).

**O que precisa ser feito**
1. Webhook do banco/PSP (pagamento confirmado) + leitura de retorno → baixa automática da parcela → evento de domínio `ParcelaPaga`.
2. Notificação em tempo real ao portal via **SignalR** (hub por condomínio/unidade) ou, no MVP, polling curto; condômino vê "pago" em segundos após o webhook.
3. Condômino: 2ª via com PDF/linha digitável/Pix copia-e-cola, acordos, taxa condominial.
4. Síndico: dashboard financeiro read-only (CP, CR, inadimplência, saldo, orçado x realizado), com permissões `*:read`.
5. **Pasta digital**: no fechamento do período (`FechamentoPeriodo`), gerar pacote do mês (balancete PDF + relação de pagamentos com comprovantes + NFs anexadas + extrato conciliado), armazenado em bucket **privado** com URL assinada; síndico/conselho acessam e podem dar "visto"/aprovar.

### 2.5 Conciliação bancária
**Já existe** — entidade `ConciliacaoBancaria`, `ConciliacaoController`, tela `conciliacao-baixa`. Porém `ProcessarArquivoConciliacaoHandler` declara não fazer parsing e o front envia nome de arquivo fixo `'extrato_maio.csv'`.

**O que precisa ser feito**
1. Upload real e parsers: **OFX** (extrato), **CNAB retorno de cobrança** (240/400) e **CNAB retorno de pagamento**; depois Open Finance/API de extrato.
2. Matching automático: retorno CNAB casa por nosso número/seu número (exato); extrato OFX casa por valor + data ± tolerância + documento; o que sobra vai para fila manual com sugestões.
3. **Débito automático** (maior dor): regras por fornecedor recorrente (concessionárias — luz, água, gás, telefone) com valor/data esperados vindos do lançamento projetado (2.6); ao ler o extrato, criar/baixar a ContaPagar automaticamente e só pedir confirmação quando a diferença passar da tolerância.
4. Conciliar efetivamente baixa a ContaPagar/Parcela (hoje só muda status da linha) e alimenta o saldo da conta bancária.

### 2.6 Lançamento projetado em CP
**Não existe** (os "agendamentos" atuais são só a ação de baixa).
**Precisa:** entidade `ContaPagarRecorrente` (fornecedor, plano de contas, valor estimado/fixo, dia, periodicidade, vigência, débito automático sim/não); job que gera lançamentos `projetado` para os próximos N meses; projetado vira real quando chega a fatura (Paybox ou conciliação do débito automático) com diferença registrada. Alimenta fluxo de caixa projetado e orçamento.

### 2.7 Previsão orçamentária
**Não existe.** (`Orcamento` no sistema é cotação de fornecedor — esse módulo está completo e é outra coisa.)
**Precisa:**
1. **Plano de contas** hierárquico por condomínio (com modelo padrão da administradora): grupos = seções do balancete. Classificar ContaPagar, Despesa e receitas por conta. Balancete passa a sair por seção.
2. `PrevisaoOrcamentaria` (condomínio, exercício, versão, status rascunho/aprovado em assembleia) com `ItemPrevisao` por conta × mês.
3. Sugestão automática a partir do realizado dos últimos 12 meses por conta + índice de reajuste (IPCA/IGP-M/dissídio) editável pelo gerente.
4. Relatório orçado × realizado por seção/mês, com desvio %; vínculo opcional com cálculo da taxa condominial (rateio).

### 2.8 Reforma tributária (EC 132/2023, LC 214/2025)
**Não existe.**
**Precisa:** campos IBS/CBS (alíquota, valor, `cClassTrib`, CST) nos documentos fiscais de entrada (ContaPagar) e saída (NFS-e da administradora); tabela de alíquotas por ano da transição (2026 teste 0,9% CBS / 0,1% IBS → 2027+ CBS cheia → 2029-2033 transição IBS); acompanhar split payment (impacta remessa e conciliação, pois o valor líquido chega ao fornecedor). Fazer junto com a NFS-e (2.9), que já precisa desses campos no leiaute novo.

### 2.9 Emissão de NFS-e (administradora → condomínios) — Padrão Nacional
**Não existe.** `CertificadoDigital` é só cadastro (sem arquivo, senha ou assinatura).
**Precisa:**
1. Cadastro fiscal da administradora como prestadora (CNPJ, IM, município, regime, código de serviço/tributação nacional).
2. Certificado A1 armazenado com segurança (bucket privado + segredo cifrado/Key Vault), validação de vencimento.
3. Geração da **DPS** (XML leiaute nacional, incluindo grupo IBS/CBS), assinatura XMLDSig, envio à API do **Sefin Nacional/ADN** (mTLS com o A1), consulta, cancelamento e evento de substituição; DANFSe PDF.
4. Faturamento: gerar NFS-e a partir da taxa de administração cobrada de cada condomínio (mensal, em lote), envio do PDF/XML ao condomínio e lançamento automático no CP do condomínio (fecha o ciclo com o Paybox).
5. Verificar se os municípios dos clientes aderiram ao emissor nacional (obrigatório para MEI; demais municípios em adesão) — se não, manter integração municipal como exceção.

---

## 3. Plano de implementação (fases)

Ordem guiada por dependência: plano de contas e fila durável destravam quase tudo; integração bancária destrava portal, conciliação e remessa.

| Fase | Entrega | Depende de | Tamanho |
|---|---|---|---|
| **0 — Correções** | Desabilitar `baixar-por-arquivo` (bug crítico); valor de guia calculado no servidor; corrigir `CondominioAdministradora.CondominioId int` vs `Guid` | — | P |
| **1 — Fundações** | Plano de contas (back + telas que hoje são stub) e classificação de lançamentos; fila durável (tabela outbox + worker, ou Hangfire em Postgres); bucket privado + cofre de segredos; e-mail com anexo | 0 | M |
| **2 — Recebíveis bancários** | Unificar ReciboRateio↔Parcela; gateway real (1 banco/PSP via API) + CNAB cobrança; PDF de boleto; envio por e-mail em lote; exportação bureau; leitura de retorno de cobrança (substitui o endpoint desabilitado) | 1 | G |
| **3 — Portal em tempo real** | Webhook de pagamento → baixa automática → SignalR; condômino com 2ª via/Pix/acordos/taxa condominial; acordo multi-parcela com juros e parcelas geradas | 2 | M |
| **4 — CP com workflow** | Novo ciclo de status; aprovação com alçada (acessor/gerente); permissão `lancar_sem_aprovacao`; fila de remessa; lançamento projetado/recorrente | 1 | M |
| **5 — Paybox** | Upload + e-mail de entrada; parse XML, código de barras, LLM com visão (OpenRouter); tela de revisão de rascunhos | 4 | M |
| **6 — Análise tributária** | Cadastro tributário de fornecedor/serviço; motor de retenção; etapa de análise no fluxo; calendário e guias automáticas | 4 | M |
| **7 — Remessa e conciliação** | CNAB 240 pagamentos + retorno; parsers OFX; matching automático; regras de débito automático usando projetados | 2, 4, 6 | G |
| **8 — Síndico e orçamento** | Dashboard read-only do síndico; pasta digital mensal no fechamento; previsão orçamentária com sugestão por histórico; orçado × realizado | 1, 7 | M |
| **9 — NFS-e Nacional + Reforma** | Cadastro fiscal, A1, DPS + assinatura + API Sefin/ADN, DANFSe, faturamento em lote da taxa de administração; campos IBS/CBS em entrada e saída | 1 | G |

Cada fase segue o padrão do repo: entidade em `Simpleto.Domain/Entities`, handler em `Simpleto.Application/Handlers/<Modulo>`, repositório Dapper em `Simpleto.Infrastructure/Repositories`, controller em `Simpleto.Api/Controllers/v1`, migration `Vnnn__*.sql` em `Simpleto.Infrastructure/Migrations` (próxima: V123), permissões em `PermissionPoliciesConfiguration.cs` + seed, telas em `Simpleto_front_adm/src/app/pages/condominio/financeiro/*`. Migrations aplicadas só com autorização explícita.

### Decisões que dependem do negócio (levantar antes da fase correspondente)
- Banco/PSP inicial para boleto e pagamentos (fase 2/7).
- `Despesa` some (vira ContaPagar) ou continua como pedido de compra (fase 4).
- Papéis exatos (acessor, gerente) e alçadas padrão por valor (fase 4).
- Layout exigido pela gráfica/bureau parceira (fase 2).
- Municípios dos clientes já no emissor nacional de NFS-e (fase 9).

---

## 4. Verificação
- **Análise:** cada status deste documento aponta arquivo/handler conferível; o bug da Fase 0 foi conferido em `ReciboRateioRepository.cs:163-184`.
- **Por fase (quando implementar):** `dotnet test` em `Simpleto.Api.Tests` (inclui `PermissionPolicyCoverageTests` para novas permissões); `npm test` no front; testes de unidade dos parsers CNAB/OFX com arquivos de exemplo dos bancos; homologação no sandbox do banco/PSP e no ambiente de produção restrita do Sefin Nacional (NFS-e); teste ponta a ponta: emitir boleto → pagar no sandbox → webhook → parcela paga no portal do condômino sem recarregar.

---

## 5. Andamento

| Fase | Status | Commits |
|---|---|---|
| 0 — Correções | ✅ Concluída em 2026-11-22 | BackApi `1790efa`, front `cebe0dd` |
| 1 — Fundações | ✅ Concluída em 2026-11-22 (V123/V124 aplicadas no Supabase) | BackApi `b472822`, front `dbf6624` |
| 2 — Recebíveis bancários | 🟡 Parcial — cálculo de boleto Bradesco/Itaú pronto; CNAB 400 remessa/retorno aguarda arquivos reais (decisão: **só CNAB**) | BackApi `cc6544f` |
| 4 — CP com workflow | ✅ Concluída em 2026-11-22 (V125/V126 aplicadas) | BackApi `fd6f6d7`, `ee80214`; front `9a52182`, `05c88a3` |

Notas da Fase 4 (decisões do usuário: unificar Despesa em Contas a Pagar; aprovador por **permissão**; 1 aprovador com alçada configurável):
- Lançamento sem `conta_pagar:lancar_sem_aprovacao` → `aguardando_aprovacao`; com a permissão → `programacao` (fila de pagamento). `conta_pagar:aprovar` concedida a AdministradoraAdmin e TenantMasterAdmin; `lancar_sem_aprovacao` a ninguém (conceder por usuário/perfil na tela de permissões).
- Alçada: reaproveita `AlcadaAprovacaoConfig` do condomínio (cada papel da faixa = 1 aprovação; sem faixa = 1). Quem lança não aprova; histórico em `aprovacoes_conta_pagar`.
- Telas: Contas a Pagar › aba **Em Aprovação** e **Recorrentes**; menu **Aprovação de Pagamentos** (`/layout/pagar/aprovacoes`).
- Despesa: criação bloqueada (aviso na tela); antigas seguem aprováveis. Balancete passa a incluir contas a pagar **pagas** no mês (valor bruto, regime de caixa).
- Recorrentes geram lançamentos `projetado` para 3 meses (worker diário); projetado só paga depois de confirmado (editado).
- Pendente para fases seguintes: notificar o aprovador (e-mail/WhatsApp) quando há lançamento novo na fila; status `aguardando_analise_tributaria` (Fase 6).

| 5 — Paybox | ✅ Concluída em 2026-11-22 (V127 aplicada) — e-mail aguarda credenciais da caixa | BackApi `1cdf6e6`, `aea2535`; front `e4d5a6b`, `ebe2125` |

Notas da Fase 5:
- Tela **Financeiro › Paybox (enviar contas)** (`/layout/pagar/paybox`): arrastar PDF/foto/XML → leitura automática → "Revisar e lançar" abre o formulário já preenchido → lançamento segue o fluxo de aprovação. Aprovador vê **Ver fatura** na fila.
- Leitura: XML NF-e/NFS-e direto; PDF com texto → linha digitável (boleto e arrecadação) e CNPJ validados por DV; IA (OpenRouter, `LlmSettings`) só para o que faltar, com validação de faixa; imagem → modelo de visão. PDF escaneado sem texto: pede envio como imagem ou preenchimento manual (não há conversão PDF→imagem).
- Fornecedor reconhecido pelo CNPJ cadastrado; condomínio pelo CNPJ do pagador (necessário para os documentos vindos por e-mail).
- **Caixa de e-mail (pendente de configuração pelo usuário)** — sem ela o worker fica parado. Configurar em user-secrets ou variáveis de ambiente (nunca no repositório):
  ```json
  "PayboxEmail": { "Caixas": [ {
      "TenantId": "<id do tenant da administradora>",
      "Host": "imap.gmail.com", "Porta": 993, "UsarSsl": true,
      "Usuario": "contas@suaadministradora.com.br", "Senha": "<senha de app>",
      "PastaEntrada": "INBOX", "PastaProcessados": "Paybox-Processados",
      "DominiosPermitidos": []
  } ] }
  ```
  Variáveis de ambiente equivalentes: `PayboxEmail__Caixas__0__Senha`, etc.

| 6 — Análise tributária | ✅ Concluída em 2026-11-22 (V128 aplicada) | BackApi `1f0bd6e`; front `bb301b5` |

Notas da Fase 6:
- **Cadastro tributário do fornecedor** (aba Fornecedores): regime (Simples, MEI, Presumido, Real, imune/isento), natureza do serviço, alíquota de ISS, reter ISS (automático/sempre/nunca) e CPRB.
- **Motor de sugestão** (`Simpleto.Domain/Services/Tributos/MotorRetencao.cs`): PIS/COFINS/CSLL 4,65%, IRRF 1% (mão de obra) ou 1,5% (profissional), INSS 11% (3,5% CPRB; Simples só no Anexo IV; base reduzida por materiais), ISS automático só nos serviços devidos no local da prestação. Dispensa federal até R$ 10,00. Pessoa física: INSS 11% sugerido; IRRF pela tabela progressiva fica manual (aviso). É **sugestão** — o analista confirma.
- **Etapa no fluxo**: o lançamento grava `analise_tributaria` = `pendente` / `conferida` / `nao_aplicavel`. Pendente **bloqueia aprovação e pagamento**. Quem tem `conta_pagar:analise_tributaria` (AdministradoraAdmin e TenantMasterAdmin) confere no próprio lançamento; os demais caem na fila **Contas a Pagar › Análise Tributária** (`/layout/pagar/analise-tributaria`). Fornecedor sem cadastro tributário → pendente. Lançamentos antigos ficaram `nao_aplicavel`.
- No formulário de lançamento: painel de sugestão ("Aplicar sugestão"), data de emissão da nota e materiais/equipamentos (dedução do INSS).
- **Guias** agora são **por condomínio** e ficam vinculadas às retenções que recolhem (a mesma retenção não entra em duas guias; cancelar a guia libera as retenções). Competência: IRRF/CSRF pela data de pagamento; INSS/ISS pela emissão da nota (sem a data, o vencimento). Só entram retenções já conferidas.
- **Calendário tributário** (aba Guias): por condomínio e competência mostra vencimento, valor a recolher, valor ainda em análise e guias emitidas; botão Gerar guia. Vencimento padrão: federais dia 20, ISS dia 10, ajustados por dias úteis/feriados nacionais (Carnaval, Sexta-feira Santa e Corpus Christi incluídos). Dia e regra (antecipar/postergar) configuráveis nos Parâmetros de Recolhimento; feriados municipais não são considerados.
- Guias antigas (anteriores à Fase 6) ficam sem condomínio e não têm retenções vinculadas.
- Fica para depois: DARF/DAM com código de barras; guia virar conta a pagar na fila de remessa (Fase 7); EFD-Reinf (a base de cálculo por retenção já é gravada e a guia lista os lançamentos).

Notas da Fase 1:
- Plano de contas: `api/v1/plano-contas` (GET/POST/PATCH/DELETE + `inicializar`), tela em Cadastros/Financeiro › Plano de Contas, plano padrão condominial em `Simpleto.Domain/Services/PlanoContasPadrao.cs`. Permissão `plano_contas:*` espelha `conta_pagar`.
- Contas a pagar ganham `plano_conta_id` (opcional), com seletor no lançamento.
- Fila durável: `IFilaTarefas` + `ITarefaHandler` + `FilaTarefasWorker` (tabela `tarefas_fila`). Ainda sem handlers — os primeiros chegam na Fase 2.
- `IEmailSenderService.EnviarAsync(..., anexos)` e `IArquivoPrivadoStorage` (Supabase, bucket privado + link assinado).
- **Fica para quando for necessário (Fase 9):** cofre de segredos para o certificado A1. Balancete por seção do plano de contas: Fase 8.
- Ainda não classificado: `despesas` (aguarda decisão da Fase 4 sobre manter ou não a entidade `Despesa`).

Notas da Fase 0:
- `baixar-por-arquivo` agora é recusado (mensagem orienta baixa manual); método de UPDATE em massa removido; botão desabilitado no front. Volta na Fase 2 com leitura real de CNAB.
- Guia de recolhimento: valor calculado no servidor (`GuiaRecolhimentoRepository.SomarRetencoesAsync`); competência valida mês 01–12.
- `CondominioAdministradora.CondominioId int`: **não alterado** — entidade legada (migração FluentMigrator SQL Server), sem uso no fluxo Postgres.
- Também corrigido: recurso `notificacao` ausente em `PermissionPoliciesConfiguration` (teste de cobertura voltou a passar; 637/637).
