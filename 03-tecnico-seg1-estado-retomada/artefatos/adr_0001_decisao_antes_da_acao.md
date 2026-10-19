---
status: aceita
date: 2026-10-19
decision-makers: arquitetura do sistema de referência (fictício)
---

# ADR 0001: gravar a decisão do modelo antes da ação que ela autoriza

> Sistema de referência da série *Arquitetura na fronteira*: o agente de atendimento de uma distribuidora B2B fictícia. Acompanha o texto "Onde fica o estado de um agente que espera o fim de semana?". Formato MADR 4.0.0, com os títulos das seções em português. O cenário de qualidade que confirma esta decisão é o CQ-02 (`cq02_aprovacao_apos_fim_de_semana.md`).

## Contexto e problema

O agente de atendimento conduz tarefas que esperam dias por eventos externos, como a aprovação do crédito comercial. O processo que executa o agente pode ser reiniciado durante a espera, por falha ou por atualização.

O próximo passo de cada tarefa é escolhido por um modelo. Duas chamadas com a mesma entrada podem produzir escolhas diferentes, e trocar o modelo ou as instruções pode mudar o comportamento sem mudar o código.

As ações externas, como registrar um pedido no ERP, usam chave de idempotência. O destino guarda a chave por prazo limitado; na documentação da Stripe, usada como referência do mecanismo, as chaves podem ser removidas depois de pelo menos 24 horas, e uma chave reutilizada depois disso gera uma requisição nova.

Como guardar o estado da tarefa para que a retomada, horas ou dias depois e talvez com outra versão das instruções, continue a partir do que já foi decidido e feito, sem repetir efeitos externos?

## Fatores de decisão

- Uma retomada não pode executar duas vezes a mesma ação externa.
- Uma decisão já tomada não pode depender de o modelo repeti-la.
- A investigação de uma execução precisa saber qual versão decidiu e com base em quais eventos.
- A tarefa precisa poder esperar dias sem processo ativo.
- Os dados pessoais citados na conversa precisam poder ser eliminados ao fim do tratamento (LGPD, art. 16), sem quebrar o registro.
- A latência de cada passo do agente.

## Opções consideradas

1. Estado em memória, gravado ao fim da execução.
2. Registro só dos fatos de negócio; na retomada, as escolhas são pedidas de novo ao modelo.
3. Registro só de acréscimo com eventos de decisão, gravados de forma assíncrona.
4. Registro só de acréscimo com eventos de decisão, intenção e resultado, cada evento de decisão e de intenção gravado de forma síncrona antes da ação.

## Resultado da decisão

Opção escolhida: **4**. É a única das quatro em que uma queda entre a escolha do modelo e a ação externa não apaga a escolha nem a intenção.

Regras que decorrem da decisão:

1. O registro da tarefa só recebe acréscimos e fica fora do processo do agente.
2. Cada escolha do modelo é gravada como evento de decisão antes da ação que ela autoriza. O evento traz:
   - a ação proposta, com a ferramenta e os argumentos;
   - a referência aos eventos que formaram o contexto daquela chamada;
   - as versões do modelo, das instruções e das ferramentas.
3. Cada ação externa grava a intenção e a chave de idempotência antes da chamada, e o resultado quando o destino o confirma.
4. Quando um processo novo sobe, o harness resolve as intenções sem resultado antes de chamar o modelo, enquanto a chave é válida. Uma intenção cuja chave venceu sem resultado vai para reconciliação com o destino, nunca para repetição.
5. A espera por um evento externo é gravada como evento de espera. O aviso externo é acrescentado ao registro e acorda a tarefa por meio do gatilho de retomada.
6. As retentativas de falha transitória pertencem à camada de execução do harness e usam a mesma chave. O modelo recebe o resultado final, com o motivo da falha. Se ele propuser de novo uma operação que já tem intenção sem resultado na mesma tarefa, a proposta é ligada a essa intenção e reaproveita a chave dela.
7. Depois de uma troca de versão, as decisões já gravadas continuam valendo, e a versão nova decide só os passos seguintes.
8. Os dados pessoais ficam fora do registro, num cadastro referenciado por identificador. Nenhum dado pessoal é usado como chave de idempotência.

### Consequências

- Bom: a retomada lê o que foi decidido, em vez de pedir de novo ao modelo.
- Bom: cada decisão pode ser investigada com a versão e os eventos que a sustentaram.
- Bom: a tarefa espera sem processo ativo, e a pendência técnica não depende da espera de negócio.
- Bom: eliminar um dado pessoal não altera o registro.
- Ruim: cada ação do agente espera uma escrita síncrona, o que aumenta a latência do passo.
- Ruim: o registro cresce com a tarefa, e a seleção do que entra no contexto de cada chamada vira um problema próprio, tratado no próximo texto da série.
- Ruim: a troca de versão durante uma tarefa exige uma política declarada antes da implantação.
- Neutro: a reconciliação de uma intenção com chave vencida depende de o destino permitir a consulta por uma referência externa. Para um ERP legado sem essa consulta, vale o que o precursor descreve sobre a camada de integração.

### Confirmação

- **Teste de interrupção:** encerrar o processo em dois pontos (entre a gravação da decisão e a chamada ao ERP; entre a chamada e a gravação do resultado) e verificar que o ERP tem um pedido só e que nenhuma escolha gravada foi pedida de novo ao modelo.
- **Teste de chave vencida:** com um ERP simulado que descarta chaves em 24 horas, deixar uma intenção sem resultado passar desse prazo e verificar que a retomada vai para reconciliação e não repete a chamada.
- **Verificação do registro:** todo evento de decisão traz as três versões e a referência ao contexto.
- **Revisão de código:** nenhuma ferramenta com efeito externo é chamada sem uma intenção gravada antes.
- **Critério de aceite:** o cenário de qualidade CQ-02.

Estes são procedimentos propostos. Nenhum resultado de execução está registrado aqui.

## Prós e contras das opções

### 1. Estado em memória, gravado ao fim da execução

- Bom: simples, sem escrita a cada passo.
- Ruim: uma queda no meio perde a tarefa. É o comportamento do modo `exit` do LangGraph, que não permite recuperar falhas no meio da execução.
- Ruim: a tarefa precisaria de um processo ativo durante toda a espera.

### 2. Registro só dos fatos de negócio

- Bom: registro menor, na forma clássica do Event Sourcing.
- Ruim: a retomada pede de novo as escolhas ao modelo, que pode escolher diferente.
- Ruim: o registro não diz qual versão decidiu nem com base em quê.

### 3. Eventos de decisão gravados de forma assíncrona

- Bom: menos latência por passo.
- Ruim: uma queda pode perder a última decisão antes da ação que ela autorizou. O modo `async` do LangGraph declara esse risco.

### 4. Eventos de decisão, intenção e resultado gravados de forma síncrona antes da ação

- Bom: cobre os fatores de decisão.
- Ruim: latência de escrita a cada ação e registro maior.

## Mais informações

- **Padrões de partida:** Event Sourcing e Retry, padrões de projeto do Azure Architecture Center. O Event Sourcing já recomenda que cada evento registre a intenção de negócio, além do estado resultante; esta decisão acrescenta a escolha do modelo e as versões de quem decidiu.
- **Alternativa para a troca de versão:** fixar a versão por tarefa. Mantém o comportamento estável até o fim da tarefa, mas exige manter mais de uma versão em execução. Reavaliar esta escolha se as instruções passarem a carregar regras de negócio, como o limite de desconto, que mudem com frequência.
- **Fontes, consultadas em 19/10/2026:**
  - Microsoft, Azure Architecture Center, "Event Sourcing pattern": https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing
  - Microsoft, Azure Architecture Center, "Retry pattern": https://learn.microsoft.com/en-us/azure/architecture/patterns/retry
  - Anthropic, "Scaling Managed Agents: Decoupling the brain from the hands", 08/04/2026, relato de engenharia do fornecedor: https://www.anthropic.com/engineering/managed-agents
  - LangGraph, "Checkpointers": https://docs.langchain.com/oss/python/langgraph/checkpointers
  - Stripe, "Idempotent requests": https://docs.stripe.com/api/idempotent_requests
  - Lei nº 13.709/2018 (LGPD), art. 16: https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709compilado.htm
  - MADR 4.0.0: https://adr.github.io/madr/
