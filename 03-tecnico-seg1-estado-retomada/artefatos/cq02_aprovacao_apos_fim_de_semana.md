# CQ-02: aprovação depois do fim de semana, com troca de versão

> Sistema de referência da série *Arquitetura na fronteira*: o agente de atendimento de uma distribuidora B2B fictícia. Acompanha o texto "Onde fica o estado de um agente que espera o fim de semana?" e o ADR 0001 (`adr_0001_decisao_antes_da_acao.md`). Segue a forma longa do cenário de qualidade da seção 10 do arc42: identificador, nome, fonte, estímulo, ambiente, artefato, resposta e medida da resposta. O CQ-01 é o cenário do prólogo (retomada com intenção pendente).
>
> A distribuidora, o pedido 4471, a chave K-7f3a e as versões v1 e v2 das instruções são fictícios. O cenário é um requisito proposto; não registra uma demonstração já executada.

## Requisito de qualidade

A retomada de uma tarefa longa continua a partir do que já foi decidido e feito, mesmo depois de dias de espera, de um reinício do processo e de uma troca de versão das instruções.

## Cenário

| Item | Conteúdo |
|---|---|
| Identificador | CQ-02 |
| Nome | Aprovação depois do fim de semana, com troca de versão |
| Fonte | O ERP, que avisa a aprovação do crédito comercial. |
| Estímulo | O aviso chega na segunda-feira às 9h a uma tarefa que espera desde sexta às 17h59. No intervalo, as instruções do agente passaram da versão v1 para a v2. |
| Ambiente | Operação normal. O processo do agente foi reiniciado na sexta às 18h por uma atualização. Quando o processo novo subiu, a intenção K-7f3a foi resolvida com a mesma chave e o pedido 4471 foi gravado no registro. Nenhum processo ficou ativo para a tarefa durante o fim de semana. |
| Artefato | O harness do agente de atendimento, o gatilho de retomada e o registro da tarefa. |
| Resposta | O gatilho acrescenta o aviso ao registro e acorda a tarefa. O harness reidrata a tarefa a partir do registro, sem pedir ao modelo nenhuma escolha já gravada. O modelo, com as instruções v2, escolhe o próximo passo, e o evento de decisão é gravado com as versões antes da ação seguinte. |
| Medida da resposta | 1. Nenhuma escolha já gravada é pedida de novo ao modelo. 2. Todo evento de decisão traz as versões do modelo, das instruções e das ferramentas. 3. Nenhuma intenção chega à segunda-feira sem resultado ou sem reconciliação. 4. O ERP tem um pedido só para a tarefa. |

## Variação: a pendência técnica esperou a aprovação

Se o processo novo não tivesse resolvido a intenção K-7f3a na sexta, ela chegaria à segunda-feira sem resultado. Com um destino que descarta chaves depois de 24 horas, como permite a documentação da Stripe, repetir a chamada com a mesma chave criaria uma requisição nova. Nesse caso, a resposta esperada é a reconciliação com o ERP antes de qualquer repetição, e a medida 4 continua valendo: um pedido só.

## Procedimento proposto para verificar

1. Criar a tarefa com um pedido de reposição e deixar o agente registrar o pedido e pedir a aprovação.
2. Encerrar o processo logo depois da chamada ao ERP, antes de a resposta ser gravada.
3. Implantar as instruções v2 e subir um processo novo; verificar que a intenção K-7f3a foi resolvida e que o modelo não foi chamado.
4. Avançar o relógio de um ERP simulado além do prazo da chave.
5. Enviar o aviso de aprovação.
6. Conferir as quatro medidas no registro e no ERP simulado.
7. Repetir sem o passo 3 (variação) e verificar que a retomada vai para reconciliação.

Nenhum resultado deste procedimento está registrado aqui.

## Fontes, consultadas em 19/10/2026

- arc42, seção 10, "Quality Requirements": https://docs.arc42.org/section-10/
- Stripe, "Idempotent requests": https://docs.stripe.com/api/idempotent_requests
