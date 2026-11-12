# PDD: Lançamento de registros de associados ativos

> Versão 1.0, baseada no fluxo n8n "Simulacao CoE". Fato = comprovado no fluxo. Definição = proposta do analista para o processo automatizado. Premissa = valor adotado para dimensionar, a validar com o demandante antes da aprovação. Credenciais do fluxo não são reproduzidas aqui.

## 1. Identificação

| Campo | Valor |
|---|---|
| **Nome do processo** | Lançamento de registros de valores de associados ativos |
| **Área demandante** | Relacionamento com associados / Financeiro da cooperativa (premissa, validar com o demandante) |
| **Autor do PDD** | Gabriel (aluno), Automação Digital · Prof. Cidi Cassol |
| **Data / versão** | 11/11/2026 · v1.0 |
| **Prioridade** | Quick win (integração simples entre duas APIs, sem regra complexa) |
| **Status** | Em revisão pelo CoE |

## 2. Objetivo e justificativa

**Objetivo:** registrar, de forma automática e padronizada, o valor de cada associado ativo na API de registros da cooperativa, eliminando o lançamento manual.

**Justificativa:** o lançamento manual associado a associado é repetitivo, sujeito a erro de digitação, omissão e duplicidade, e não deixa trilha de auditoria. A automação reduz esforço e retrabalho e permite conferência posterior.

**Resultado esperado:** 100% dos associados ativos da consulta com registro criado uma única vez, com log por execução e falhas sinalizadas.

**Fato no fluxo:** o fluxo atual já consulta a API de associados (status=ativo) e faz POST em /registros com nome e valor. Faltam tratamento de erro, idempotência, logs e gestão segura de credenciais, tratados nas seções 5, 6 e 10.

## 3. Processo atual (AS IS), passo a passo

**Processo manual (AS IS), reconstruído a partir do fluxo (premissa, validar em entrevista):**

1. O responsável acessa a base ou tela de associados e filtra os que estão com status ativo.
2. Para cada associado, anota o nome e o valor a registrar.
3. Acessa o sistema de registros e lança manualmente associado e valor, um a um.
4. Confere visualmente se todos foram lançados e comunica a conclusão.

**Processo automatizado (TO BE), conforme o fluxo (fato):**

1. Disparo manual da execução pelo responsável.
2. Consulta à API de associados com status = ativo.
3. Para cada associado retornado, usa-se nome e valor.
4. POST em /registros com associado = nome e valor = valor.
5. Encerramento da execução.

**Melhorias definidas para a versão de produção:** verificar duplicidade antes do POST, validar a resposta de cada POST, gerar log e resumo final da execução.

## 4. Regras de negócio

**RN01 (fato):** só associados com status = ativo são processados.

**RN02 (definição):** cada associado deve ter no máximo um registro por execução/competência; reprocessar não pode duplicar lançamentos.

**RN03 (definição):** associado sem nome ou com valor ausente, não numérico ou negativo não é registrado e vai para a lista de exceções.

**RN04 (definição):** o valor é enviado exatamente como recebido da API, sem recálculo pelo robô (fato: o fluxo não altera o valor).

**RN05 (premissa, validar):** a execução ocorre uma vez por competência, em dia útil e fora do horário de pico do sistema de registros.

**Nota técnica:** os nós "dados" e "fazer" (x=ok) são passos técnicos e não geram regra de negócio.

## 5. Exceções e como tratar

**Situação atual (fato):** o fluxo não trata exceções. Qualquer falha interrompe a execução e deixa processamento parcial sem registro de quais associados foram enviados.

**Tratamento definido para a versão de produção:**

- **Lista de associados vazia (negócio):** encerrar sem erro, registrar em log e avisar o responsável.
- **Nome ou valor inválido (negócio):** pular o item, incluir na lista de exceções do relatório final, sem interromper o lote.
- **Associado já registrado (negócio):** não reenviar; registrar como "já processado".
- **API indisponível ou timeout (sistema):** até 3 tentativas com espera crescente; persistindo, interromper e notificar o responsável e o time de automação.
- **401/403 (sistema):** interromper imediatamente e notificar; não repetir.
- **Erro 4xx/5xx no POST /registros (sistema):** tentar novamente em caso de 5xx; em 4xx, registrar o item como falha e seguir.
- **Falha no meio do lote (sistema):** a execução seguinte deve retomar apenas os itens pendentes, graças ao controle de itens já processados.

**Parâmetros (3 tentativas, canal de notificação):** premissas, ajustar com o demandante.

## 6. Sistemas, dados e LGPD

**Sistemas envolvidos**

**API da cooperativa** (api.exemplo-cooperativa.com): GET /associados e POST /registros (fato).

**n8n** como orquestrador (fato).

**Banco de dados** (base "coop") citado em string de conexão no nó "dados", mas não usado por nenhum nó (fato). Definição: o banco só entra no processo se houver controle de itens processados (seção 5); caso contrário, remover a conexão do fluxo.

**Entradas (dados)**

Lista de associados ativos retornada pela API; campos usados: nome e valor (fato).

**Saídas (dados)**

Registro criado em /registros com associado e valor (fato). Definição para produção: log por execução (total lido, registrado, ignorado, com falha) e relatório de exceções.

**Dados pessoais / sensíveis**

**Dados pessoais (fato):** nome do associado. O valor vinculado ao nome é dado financeiro confidencial.

**Credenciais expostas no fluxo (fato):** chave de API na URL do nó HTTP Request1 e usuário, senha e string de conexão do banco em texto claro no nó "dados". Não reproduzidas neste documento.

**Proteção definida:** mover todas as credenciais para o cofre/Credentials do n8n; rotacionar as expostas; remover o nó "dados"; restringir acesso ao fluxo e ao histórico de execuções; limitar a retenção do histórico; mascarar nome e valor nos logs.

**LGPD:** tratamento limitado ao necessário (minimização: só nome e valor); finalidade e base legal a registrar pelo demandante e pelo encarregado (DPO); validar com compliance o sigilo das operações financeiras. Transporte via HTTPS (fato).

## 7. Volume e frequência

**Volume**

**Premissa de dimensionamento:** 1.000 associados ativos por execução (valor ilustrativo, substituir pelo real).

**Frequência**

**Definição:** mensal, uma execução por competência, com disparo agendado em produção. Hoje o fluxo é manual (fato).

**Janela**

**Premissa:** dia útil, entre 22h e 6h, fora do horário de atendimento; a duração deve ser medida no piloto.

## 8. Benefício esperado

**Cálculo ilustrativo (premissas, validar):** 1.000 associados × 2 min por lançamento manual = 2.000 min ≈ 33 h/mês, ou cerca de 400 h/ano de esforço manual que passam a ser uma execução automática com revisão de exceções.

**Benefícios qualitativos:** menos erros de digitação e omissão, sem duplicidade, trilha de auditoria por execução e padrão reaproveitável pelo CoE.

**Para o business case final:** substituir as premissas por tempo e volume medidos e subtrair o custo de desenvolvimento e manutenção para chegar ao ROI.

## 9. Critérios de aceitação (Definition of Done)

- O robô cobre o caminho feliz e as exceções listadas.
- Credenciais no cofre, nada escrito no fluxo.
- Logs e trilha de auditoria da execução.
- PDD e SDD entregues e revisados pelo CoE.

## 10. Riscos e dependências

**Riscos (fato, observados no fluxo atual)**

1. **Alto:** credenciais em texto claro. Mitigação: cofre/Credentials, rotação, remover nó "dados".
2. **Alto:** sem idempotência; reexecução duplica lançamentos. Mitigação: RN02 e controle de itens processados.
3. **Alto:** sem tratamento de erro; processamento parcial. Mitigação: seção 5.
4. **Médio:** sem log, auditoria ou notificação; x=ok aparece mesmo sem validar o POST. Mitigação: validar resposta e gerar relatório.
5. **Médio:** sem paginação na consulta. Mitigação: paginar ou conferir o total retornado.
6. **Médio:** dados pessoais no histórico de execuções. Mitigação: retenção curta, acesso restrito, mascaramento.
7. **Baixo:** nomes de nós genéricos ("node", "dados", "fazer", "HTTP Request1/2"). Mitigação: padrão de nomenclatura do CoE.
8. **Baixo:** disparo manual depende de uma pessoa. Mitigação: agendamento.

**Dependências:** disponibilidade e contrato das APIs; credenciais em cofre; validação de LGPD/compliance; definição do demandante sobre as premissas.

## 11. Premissas e perguntas pendentes

**Premissas adotadas (validar antes da aprovação)**

1. Área demandante e AS IS manual (seção 3, passos manuais).
2. "valor" é monetário e vem pronto da API.
3. A resposta de /associados é uma lista com nome e valor por item.
4. Volume (1.000), frequência mensal, janela noturna, 2 min por lançamento e 3 tentativas de retry.
5. O banco de dados citado não é necessário ao processo, salvo para o controle de itens processados.

**Perguntas pendentes ao demandante**

1. O que o "valor" representa e quem o define?
2. Qual a periodicidade real, o prazo limite e o volume por execução?
3. Como identificar um registro já feito (chave única) para evitar duplicidade?
4. Quem deve ser notificado em falha e por qual canal?
5. Quais outros dados do associado a API retorna e qual a base legal do tratamento?
