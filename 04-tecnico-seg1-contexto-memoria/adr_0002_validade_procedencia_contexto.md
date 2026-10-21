---
status: aceita
date: 2026-10-20
decision-makers: arquitetura do sistema de referência (fictício)
---

# ADR 0002: cada item do contexto leva fonte, momento da leitura e validade

> Sistema de referência da série *Arquitetura na fronteira*: o agente de atendimento de uma distribuidora B2B fictícia. Acompanha o texto "Quanto tempo vale o estoque que o agente leu na sexta?". Formato MADR 4.0.0, com os títulos das seções em português. Estende as regras 2 e 10 do ADR 0001 (`../../03-tecnico-seg1-estado-retomada/artefatos/adr_0001_decisao_antes_da_acao.md`). O que a memória do cliente guarda é decidido no ADR 0003 (`adr_0003_memoria_valor_ou_endereco.md`). O cenário de qualidade que confirma as duas decisões é o CQ-03 (`cq03_estoque_de_sexta_oferta_de_segunda.md`).
>
> A distribuidora, as luvas nitrílicas, as 120 caixas, o desconto de 8% e os prazos de validade abaixo são fictícios e ilustrativos.

## Contexto e problema

A cada chamada, o harness do agente de atendimento monta o contexto do modelo a partir de três lugares: o registro da tarefa (ADR 0001), a memória do cliente (ADR 0003) e leituras feitas nos sistemas de origem, como o estoque, os contratos e o crédito comercial no ERP.

O registro só recebe acréscimos e guarda o resultado de cada consulta como evento. Esse resultado é uma cópia da origem, tirada num momento, sem prazo de validade. Quando o harness transforma o evento em texto, a idade da informação se perde: o modelo lê o estoque de sexta como se fosse de agora.

No Cache-Aside, o código percebe a ausência de um item no cache e lê na fonte. No agente, se um dado não está no contexto, nenhum código percebe a falta, e o modelo pode seguir sem ele.

A janela de contexto é limitada, e o relato da Anthropic sobre engenharia de contexto descreve a queda da capacidade de recuperar informação à medida que o número de tokens cresce (*context rot*). Cada token tem custo. Um resumo produzido por compactação parafraseia valores, e a invalidação por chave não alcança um valor parafraseado.

Como montar o contexto de cada chamada para que nenhum compromisso com o cliente se apoie numa cópia vencida, para que o modelo saiba o que ficou de fora e para que uma investigação saiba o que o modelo leu e com que idade?

## Fatores de decisão

- Nenhuma oferta, prazo de entrega, preço ou condição comercial chega ao cliente apoiada em cópia vencida.
- A regra de validade não pode depender de o modelo desconfiar de dados antigos, porque ela seria cumprida em algumas execuções e em outras não.
- O modelo precisa saber que uma informação fora do contexto existe e onde buscá-la.
- A investigação de uma decisão precisa saber quais itens o modelo leu, de que fonte e com que idade.
- O tamanho do contexto e o custo de cada chamada.
- A latência de cada passo e a carga sobre os sistemas de origem.
- O aproveitamento do cache de prompt, que depende de um início de entrada estável.

## Opções consideradas

1. Histórico inteiro no contexto até o limite da janela, com corte dos eventos mais antigos.
2. Compactação por resumo feito pelo modelo, sem referências aos eventos e às fontes resumidos.
3. Recuperação por similaridade (busca com embeddings) sobre todo o histórico e as fontes, sem validade nem procedência nos itens.
4. Itens com metadados de fonte, leitura, validade e escopo; validade aplicada pelo harness; referência para o que fica fora; leitura na fonte antes de cada compromisso com o cliente.

## Resultado da decisão

Opção escolhida: **4**. É a única das quatro em que o harness, e não o modelo, sabe a idade de cada item e impede que um compromisso se apoie numa cópia vencida.

Regras que decorrem da decisão:

1. **Metadados.** Todo item que entra no contexto traz, além do conteúdo:
   - a fonte: o dono do dado, o sistema de origem e o endereço do dado nesse sistema;
   - o momento da leitura;
   - a validade, por tempo ou até um evento de invalidação;
   - o escopo, que é o cliente da tarefa.

   Um item sem esses metadados não entra.
2. **Política de validade.** Cada tipo de dado tem uma validade declarada pelo dono do dado e mantida pela arquitetura. Na distribuidora (valores ilustrativos):

   | Dado | Dono e origem | Validade |
   |---|---|---|
   | Estoque por item | Logística, ERP | 15 minutos |
   | Preço por cliente e desconto do contrato | Comercial, ERP | Até o próximo evento de tabela ou de contrato do cliente; sem eventos, 24 horas |
   | Limite de crédito | Financeiro, ERP | Nenhuma cópia vale para um compromisso: é lido na fonte a cada uso |
   | Preferência do comprador | Conversa, memória do cliente (ADR 0003) | Até ser substituída por um item novo |

3. **Aplicação pelo harness.** Ao montar o contexto, o harness compara o momento da leitura com a validade. Os itens vencidos que o tipo de tarefa declara como necessários são lidos de novo na fonte. Os demais entram marcados como vencidos, com o endereço para uma nova leitura.
4. **Compromissos por ferramenta.** Oferta, prazo de entrega, preço e condição comercial chegam ao cliente por uma ferramenta com argumentos estruturados. Antes de executá-la, o verificador de compromissos lê na fonte os dados de que ela depende, mesmo que o contexto tenha uma cópia válida. Se a fonte diverge dos argumentos, a ferramenta recusa e devolve o motivo ao modelo. A regra estende a esses compromissos a orientação do Cache-Aside para dados sensíveis: ler sempre da fonte primária.
5. **Texto livre.** As instruções proíbem compromissos em texto livre. O verificador de compromissos retém uma mensagem ao cliente que cite quantidade disponível, prazo ou preço sem uma chamada da ferramenta correspondente no mesmo passo.
6. **Referência para o que fica fora.** O contexto traz a lista do que ficou de fora, com o identificador, o endereço e uma linha de descrição de cada item, para o modelo carregar o que precisar por ferramenta.
7. **Seleção.** Sob o orçamento de tokens de cada chamada [P], sai primeiro o item menos útil para a próxima decisão. O resultado bruto de uma chamada de ferramenta antiga sai do contexto, e o endereço da leitura fica.
8. **Resumo.** O resumo usado no contexto é gravado como evento (ADR 0001, regra 10) com a lista dos eventos e das fontes de que deriva. Ele não copia valores que têm validade: cita o item pela referência, e o valor entra no contexto pela regra 3.
9. **Ordem de montagem.** Do mais estável ao mais volátil: instruções e descrições das ferramentas; memória do cliente; resumo e eventos da tarefa depois dele, com as intenções pendentes; lista do que ficou de fora; itens lidos nas fontes. O cache de prompt só aproveita o início idêntico da entrada, e um item relido no meio impediria o aproveitamento de tudo o que vem depois.
10. **Evento de decisão.** Além do que a regra 2 do ADR 0001 exige, o evento de decisão lista os itens do contexto com a fonte, o momento da leitura, a validade e a indicação de vencido.
11. **Escopo.** Itens de outro cliente nunca entram no contexto da tarefa. As regras de acesso por usuário e por papel ficam para as decisões do tema autoridade.

### Consequências

- Bom: nenhum compromisso com o cliente se apoia numa cópia vencida, independentemente do que o modelo decidir.
- Bom: o modelo sabe que um dado fora do contexto existe e onde buscá-lo.
- Bom: a investigação de uma decisão sabe o que o modelo leu, de que fonte e com que idade.
- Bom: o resumo mantém o caminho de volta à origem, e a mudança de um valor na fonte não fica presa numa paráfrase.
- Bom: a ordem de montagem preserva o aproveitamento do cache de prompt.
- Ruim: a leitura na fonte antes de cada compromisso aumenta a latência do passo e a carga sobre o ERP.
- Ruim: os metadados ocupam tokens em cada item.
- Ruim: a política de validade precisa de dono e de manutenção para cada tipo de dado.
- Ruim: a regra depende de endereços estáveis nos sistemas de origem.
- Neutro: a validade por evento depende de o sistema de origem publicar eventos de mudança. Sem eles, vale o prazo por tempo.
- Neutro: a retenção de mensagens com valores em texto livre (regra 5) é uma barreira por padrão de texto e pode deixar passar uma formulação que ela não reconhece. A medida da eficácia fica para as avaliações do tema evolução em produção.

### Confirmação

- **Teste de item vencido:** gravar uma consulta de estoque, avançar o relógio além da validade e verificar que o item entra marcado como vencido, com o endereço, ou é relido, conforme o tipo de tarefa.
- **Teste de compromisso:** com uma cópia válida de estoque no contexto e a fonte alterada depois da leitura, fazer o modelo propor uma oferta e verificar que o verificador lê a fonte, que a ferramenta recusa e que nenhuma mensagem com o valor antigo chega ao cliente.
- **Teste de texto livre:** induzir o modelo a citar preço ou quantidade numa resposta sem a ferramenta e verificar que a mensagem é retida.
- **Teste de resumo:** compactar uma conversa que cita o desconto, mudar o desconto na fonte e verificar que o resumo traz a referência, e não o valor.
- **Teste de referência:** deixar um item fora do contexto e verificar que a lista traz o endereço e que o modelo consegue carregá-lo pela ferramenta.
- **Teste de ordem:** comparar o início da entrada de duas chamadas seguidas da mesma tarefa e verificar que a releitura de um item não altera nada antes da camada dos itens lidos nas fontes.
- **Verificação do registro:** todo evento de decisão lista os itens do contexto com os metadados da regra 1.
- **Critério de aceite:** o cenário de qualidade CQ-03.

Estes são procedimentos propostos. Nenhum resultado de execução está registrado aqui.

## Prós e contras das opções

### 1. Histórico inteiro até o limite da janela

- Bom: simples; nada é escolhido.
- Ruim: o resultado de uma consulta antiga entra sem idade, e o modelo o lê como atual.
- Ruim: o custo e o *context rot* crescem com a tarefa, e o corte dos eventos mais antigos pode tirar uma intenção pendente ou uma decisão que a próxima chamada precisa ver.

### 2. Resumo do modelo sem referências

- Bom: contexto pequeno.
- Ruim: o resumo parafraseia valores com validade, e nenhuma chave leva até eles quando a fonte muda.
- Ruim: uma informação que entrou só pelo resumo perde o caminho de volta à origem, e a compactação agressiva pode perder um detalhe importante.

### 3. Recuperação por similaridade sem validade nem procedência

- Bom: escala para históricos e fontes grandes.
- Ruim: a similaridade não diz se o item ainda vale nem de onde ele veio.
- Ruim: o que não foi recuperado continua invisível para o modelo.

### 4. Itens com metadados, validade no harness e leitura na fonte antes de cada compromisso

- Bom: cobre os fatores de decisão.
- Ruim: latência e carga nos sistemas de origem, metadados em cada item e política de validade a manter.

## Mais informações

- **Padrões de partida:** Cache-Aside, padrão de projeto do Azure Architecture Center (carga sob demanda, invalidação na escrita, expiração ajustada ao padrão de acesso, consistência não garantida, dados sensíveis lidos da fonte primária); e dados como produto, princípio do data mesh (Dehghani), de onde vêm o dono, o endereço e a procedência dos metadados. Dehghani formulou o princípio para dados analíticos; a aplicação a dados operacionais é extensão desta decisão.
- **Vocabulário de procedência:** os metadados correspondem a conceitos do W3C PROV-DM (Recomendação de 30/04/2013). O item é uma *entity*; a leitura na fonte é a *generation* da cópia; o dono é a *attribution*; o fim da validade é a *invalidation*; o resumo que deriva de eventos é uma *derivation*. A correspondência é do autor e serve para quem quiser serializar os metadados num formato padronizado.
- **Recuperação antecipada e sob demanda:** o relato da Anthropic descreve a recuperação antes da chamada, por embeddings, e a recuperação *just in time*, com identificadores leves e ferramentas, e considera que os agentes mais eficazes podem combinar as duas. As regras 3, 4 e 6 são a combinação adotada na distribuidora.
- **Relação com o ADR 0001:** a regra 10 deste ADR estende a regra 2 do ADR 0001 (conteúdo do evento de decisão), e a regra 8 estende a regra 10 (resumo gravado como evento).
- **Fontes, consultadas em 19 e 20/10/2026:**
  - Microsoft, Azure Architecture Center, "Cache-Aside pattern": https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside
  - Zhamak Dehghani, "How to Move Beyond a Monolithic Data Lake to a Distributed Data Mesh", 20/05/2019: https://martinfowler.com/articles/data-monolith-to-mesh.html
  - Zhamak Dehghani, "Data Mesh Principles and Logical Architecture", 03/12/2020: https://martinfowler.com/articles/data-mesh-principles.html
  - Anthropic, "Effective context engineering for AI agents", 29/09/2025, relato de engenharia do fornecedor: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
  - W3C, "PROV-DM: The PROV Data Model", Recomendação de 30/04/2013: https://www.w3.org/TR/prov-dm/
  - Documentação de cache de prompt: Anthropic, https://platform.claude.com/docs/en/build-with-claude/prompt-caching; Google Cloud, https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/context-cache/context-cache-overview; OpenAI, https://developers.openai.com/api/docs/guides/prompt-caching
  - MADR 4.0.0: https://adr.github.io/madr/
