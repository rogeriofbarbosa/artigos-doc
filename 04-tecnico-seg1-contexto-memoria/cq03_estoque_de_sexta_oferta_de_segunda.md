# CQ-03: estoque de sexta, oferta de segunda

> Sistema de referência da série *Arquitetura na fronteira*: o agente de atendimento de uma distribuidora B2B fictícia. Acompanha o texto "Quanto tempo vale o estoque que o agente leu na sexta?", o ADR 0002 (`adr_0002_validade_procedencia_contexto.md`) e o ADR 0003 (`adr_0003_memoria_valor_ou_endereco.md`). Segue a forma longa do cenário de qualidade da seção 10 do arc42: identificador, nome, fonte, estímulo, ambiente, artefato, resposta e medida da resposta. O CQ-01 é o cenário do prólogo, e o CQ-02, o do texto anterior.
>
> A distribuidora, as luvas nitrílicas, as 120 caixas, o desconto de 8%, o pedido 4471 e as versões v1 e v2 das instruções são fictícios. O cenário é um requisito proposto; não registra uma demonstração já executada.

## Requisito de qualidade

Nenhum compromisso com o cliente se apoia numa cópia vencida, e cada decisão registra o que o modelo leu e com que idade.

## Cenário

| Item | Conteúdo |
|---|---|
| Identificador | CQ-03 |
| Nome | Estoque de sexta, oferta de segunda |
| Fonte | O modelo, na retomada da tarefa acordada pelo aviso de aprovação do crédito. |
| Estímulo | Na segunda-feira às 9h, com as instruções v2, o modelo propõe oferecer ao comprador as luvas nitrílicas que ele consultou na sexta às 17h58. |
| Ambiente | Operação normal. O registro da tarefa tem o resultado da consulta de sexta (120 caixas). O estoque zerou no sábado. O desconto do cliente mudou em setembro no ERP, e a memória do cliente guarda o endereço do contrato. Uma conversa de agosto registra que o comprador citou um desconto de 8%. |
| Artefato | O harness do agente de atendimento (coordenador do passo, política de validade, leitor de fontes e verificador de compromissos) e a memória do cliente. |
| Resposta | O coordenador do passo, ao montar o contexto, marca o estoque de sexta como vencido, com o endereço da consulta, e põe o desconto como endereço do contrato. O modelo propõe a oferta pela ferramenta de oferta. Antes de executá-la, o verificador de compromissos lê o estoque e o desconto no ERP. Como o estoque está zerado, o verificador recusa a oferta e devolve o motivo ao coordenador, que o repassa ao modelo, que informa o comprador com os dados de segunda-feira. O evento de decisão lista os itens do contexto com fonte, momento da leitura, validade e indicação de vencido. |
| Medida da resposta | 1. Nenhuma oferta, prazo ou preço chega ao cliente sem uma leitura na fonte feita depois da proposta do modelo. 2. Todo item do contexto traz fonte, momento da leitura, validade e escopo. 3. O evento de decisão lista esses itens. 4. A memória do cliente não tem o valor de nenhum campo do mapa de fatos com dono. 5. Nenhuma mensagem ao comprador cita quantidade disponível, prazo ou preço sem uma chamada da ferramenta de oferta no mesmo passo. |

## Variação: o comprador repete o desconto antigo

Na mesma conversa, o comprador escreve "com o nosso desconto de 8%". A resposta esperada: o agente usa o desconto lido no ERP, a memória não grava o valor, e a divergência é registrada como evento da tarefa e encaminhada ao vendedor responsável pelo cliente (ADR 0003, regra 4). As medidas 1 e 4 continuam valendo.

## Procedimento proposto para verificar

1. Criar a tarefa de sexta com a consulta de estoque das 17h58 e o pedido 4471, com as instruções v1.
2. Gravar na memória do cliente o endereço do contrato e alterar o desconto no ERP simulado.
3. Zerar o estoque das luvas no ERP simulado e avançar o relógio até segunda às 9h.
4. Implantar as instruções v2 e enviar o aviso de aprovação.
5. Conferir o contexto montado (itens, metadados, ordem) e o evento de decisão.
6. Conferir que a ferramenta de oferta recusou e que nenhuma mensagem com 120 caixas ou com 8% chegou ao comprador simulado.
7. Repetir com a variação e conferir a memória e o encaminhamento ao vendedor.

Nenhum resultado deste procedimento está registrado aqui.

## Fontes, consultadas em 19 e 20/10/2026

- arc42, seção 10, "Quality Requirements": https://docs.arc42.org/section-10/
- Microsoft, Azure Architecture Center, "Cache-Aside pattern": https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside
