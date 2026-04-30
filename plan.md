## Plan: Monitor Financeiro Multiusuário

Construir um monitor financeiro web com frontend React e backend FastAPI + SQLite, orientado a ingestão de extratos em múltiplas pastas locais por pessoa, classificação híbrida (regras visuais + regex), detecção de transferências, suporte a câmbio (API + ajuste manual), entradas manuais via planilhas e gestão de saldo devedor contemplando parcelas em aberto de cartão e dívidas manuais importadas por planilhas.

**Steps**
1. Fase 0 - Definição técnica e bootstrap do produto (bloqueia todas as demais fases).
2. Fase 1 - Fundação de domínio, segurança e motor DAG: autenticação multiusuário familiar; modelo de dados principal (pessoas, contas, transações, regras, categorias); trilha de auditoria de importação; implementação do motor DAG de transformações — cada nó do grafo representa uma base intermediária materializada em SQLite, cada aresta representa dependência de recálculo; ao alterar qualquer base fonte, o motor identifica o subgrafo downstream impactado e reprocessa apenas os nós afetados em ordem topológica.
3. Fase 2 - Ingestão MVP e nó RAW da DAG (paralelo interno entre parsers): CSV e XLSX como fluxos principais + pipeline de OCR para PDF; normalização, validação e deduplicação por hash canônico; cada arquivo importado gera ou atualiza o nó RAW da conta correspondente na DAG e dispara propagação downstream. Depende da Fase 1.
4. Fase 3 - Classificação MVP e nó CLASSIFIED da DAG (paralelo interno entre motor e UI): motor de regras com precedência, interface visual com regex avançado, fila de não classificadas e classificação manual em lote com preview de impacto; resultado materializado no nó CLASSIFIED; alteração de regra ou classificação manual invalida e reprocessa apenas o subgrafo a partir desse nó. Depende da Fase 2.
5. Fase 4 - Relações entre transações e nó LINKED da DAG: modelo genérico N:M de relações inter-transação via tabela `transaction_links(id, source_tx_id, target_tx_id, link_type, confidence, metadata_json)` no SQLite, com os tipos: `transfer` (1:1, sinais opostos), `payment_detail` (1:N, pagamento ↔ detalhamento de produtos/itens de outra fonte), `installment` (N:1, parcelas → compra original), `reimbursement` (N:M, despesas ↔ reembolsos parciais), `invoice_payment` (1:N, pagamento de fatura ↔ lançamentos do extrato); resolvers plugáveis por tipo — o resolver de transferência é apenas um deles; matching automático gera links com score de confiança, casos abaixo do limiar vão para fila de revisão manual; resultado materializado no nó LINKED (renomeado de RECONCILED); invalidação pontual ao alterar um link afeta apenas as transações participantes e seu downstream. Depende da Fase 3.
6. Fase 5 - Câmbio, FIFO e nó FX da DAG: três camadas com precedência explícita: (a) taxa explícita do extrato/planilha — imutável após confirmação; (b) taxa ausente — calculada por custo médio ponderado FIFO por par moeda/conta a partir do nó LINKED, materializada no nó FX_LOTS; alteração em qualquer compra de moeda invalida o nó FX_LOTS e seus downstream; (c) saldo remanescente avaliado a mercado via API externa, materializado no nó FX_MARKET — override manual atualiza apenas esse nó sem propagar upstream. Consolidação em moeda base por pessoa e global familiar lê do nó FX_MARKET. Depende da Fase 4.
7. Fase 6 - Cartão de crédito, dívidas manuais, saldo devedor e nó LIABILITIES da DAG: modelagem de parcelamento, geração/ligação de parcelas, ingestão de dívidas manuais via planilhas; cálculo de saldo devedor consolidado (parcelas em aberto + dívidas manuais) materializado no nó LIABILITIES; baixa de quitação invalida apenas esse nó e seus downstream. Depende da Fase 2.
8. Fase 7 - Qualidade de uso diário: ocultar/exibir transações (flag no nó RAW propaga invalidação downstream), monitor dedicado de não classificadas, atalhos de classificação rápida, filtros avançados por pessoa/conta/moeda/período/categoria. Depende das Fases 3 e 4.
9. Fase 8 - Testes e readiness: suíte de testes unitários/integrados/E2E incluindo testes de propagação DAG (alterar nó fonte → verificar quais nós são invalidados e reprocessados); testes de performance de ingestão e consulta; observabilidade do estado de cada nó (válido/inválido/em reprocessamento); estratégia de backup/restore por snapshot dos nós materializados. Depende de todas as fases anteriores.

**Relevant files**
- Workspace atualmente vazio: não há arquivos existentes para reaproveitamento direto.
- Estrutura alvo a criar na execução:
  - backend/app/main.py
  - backend/app/api/
  - backend/app/models/
  - backend/app/services/dag/ — motor DAG: definição de nós, resolução de dependências, invalidação e reprocessamento em ordem topológica
  - backend/app/services/ingestion/
  - backend/app/services/classification/
  - backend/app/services/linking/ — resolvers plugáveis por tipo de relação (transfer, payment_detail, installment, reimbursement, invoice_payment)
  - backend/app/services/fx/
  - backend/alembic/
  - frontend/src/
  - frontend/src/pages/
  - frontend/src/features/transactions/
  - frontend/src/features/rules/
  - frontend/src/features/imports/
  - frontend/src/features/reconciliation/

**Verification**
1. Ingestão: importar lotes mistos CSV/XLSX/PDF (OCR), com registro de erros por linha e deduplicação idempotente ao reimportar os mesmos arquivos.
2. Classificação: validar precedência de regras (manual > transferência > padrão > histórico), preview obrigatório em ações em lote e taxa de não classificadas reduzindo após criação de regras.
3. Relações entre transações: validar cada tipo de link — transfer (sinais opostos iguais), payment_detail (soma dos detalhes ≈ valor do pagamento), installment (soma das parcelas = compra original), reimbursement (N:M com rastreabilidade parcial), invoice_payment; casos abaixo do limiar de confiança devem ir para revisão e nunca auto-confirmar; alterar um link invalida apenas as transações participantes downstream.
4. Câmbio: confirmar que taxa explícita do extrato é imutável após confirmação; validar cálculo FIFO — simular série de compras de moeda seguida de gastos parciais e conferir que cada gasto usa o custo das unidades mais antigas; validar que saldo remanescente a mercado usa cotação da API; garantir que override de cotação de mercado não afeta taxas históricas nem o custo FIFO calculado.
5. Saldo devedor: conferir cálculo consolidado de saldo devedor (parcelas em aberto de cartão + dívidas manuais de planilhas), incluindo baixa por quitação parcial/total e rastreabilidade da origem da dívida.
6. UX operacional: ocultar/desocultar transações sem perda de histórico; painel de não classificadas com filtros e classificação em lote funcional.
7. Performance e DAG: alterar uma compra de moeda → confirmar que apenas FX_LOTS e downstream são reprocessados (RAW e CLASSIFIED permanecem intactos); medir tempo de reprocessamento parcial vs. reprocessamento total.
- DAG: estado de cada nó (válido/inválido/em reprocessamento) visível via API e na interface de administração.

**Decisions**
- Escopo funcional confirmado:
  - Multiusuário com login por pessoa/família.
  - Formatos prioritários no MVP: CSV, XLSX e PDF com OCR.
  - Regras por interface visual com suporte a regex avançado.
  - Câmbio: (a) taxa explícita do extrato — imutável; (b) taxa ausente no extrato — calculada por custo médio FIFO das compras de moeda que geraram o saldo, consumindo lotes mais antigos primeiro; (c) saldo remanescente avaliado a mercado via API externa, com override manual permitido apenas nessa camada.
  - Parcelas de cartão no MVP somente para compras parceladas.
  - Saldo devedor deve consolidar parcelas em aberto e dívidas manuais inseridas via planilhas.
- Arquitetura DAG com stack híbrida: nós OLTP (users, accounts, rules, import_logs, dag_node_status, manual_overrides, transaction_links, fx_rates_daily) em **SQLite**; nós analíticos materializados (RAW, CLASSIFIED_AUTO, CLASSIFIED, LINKED, FX_LOTS, FX_MARKET, LIABILITIES) em **DuckDB** (columnar, OLAP, licença MIT, embedded sem servidor). Queue de reprocessamento via asyncio + FastAPI background tasks (sem Redis/Celery).
- Topologia DAG confirmada:
  RAW → CLASSIFIED_AUTO → CLASSIFIED (merge com MANUAL_OVERRIDES) → LINKED → FX_LOTS → FX_MARKET
  RAW → LIABILITIES (branch independente)
  MANUAL_OVERRIDES (SQLite, durável) alimenta CLASSIFIED via merge — nunca é reprocessado pelo motor de regras.
- Relações inter-transação: modelo genérico N:M via `transaction_links` no SQLite com tipos `transfer`, `payment_detail`, `installment`, `reimbursement`, `invoice_payment`; resolvers plugáveis por tipo; transferência é um resolver entre outros; nó LINKED (anteriormente RECONCILED) materializa o resultado em DuckDB.
- Dois tipos de invalidação DAG: (1) estrutural — regra alterada ou nova importação reprocessa CLASSIFIED_AUTO + re-merge + downstream completo; (2) pontual — override manual re-merge apenas as linhas afetadas + downstream dessas linhas.
- Cotações de câmbio: apenas fechamento diário (um registro por par de moedas por data) armazenado em `fx_rates_daily` no SQLite. TTL simples: se a data de hoje já existe na tabela, não chama a API. Sem cache in-process, sem intraday.
- Transferências ambíguas ficam explicitamente fora de auto-confirmação no MVP.

**Further Considerations**
1. OCR de PDF no MVP: começar com escopo controlado (layouts mais comuns) e ampliar por templates conforme arquivos reais para evitar atrasos.
2. SQLite em cenário multiusuário familiar: adequado para início local; prever migração para Postgres sem reescrever domínio.
3. Regras avançadas: versionar alterações de regras para permitir rollback quando uma regra ampla classificar transações incorretamente.