---
status: aceita
date: 2026-10-20
decision-makers: arquitetura do sistema de referência (fictício)
---

# ADR 0003: a memória guarda o valor só dos fatos sem sistema de origem

> Sistema de referência da série *Arquitetura na fronteira*: o agente de atendimento de uma distribuidora B2B fictícia. Acompanha o texto "Quanto tempo vale o estoque que o agente leu na sexta?". Formato MADR 4.0.0, com os títulos das seções em português. Complementa o ADR 0002 (`adr_0002_validade_procedencia_contexto.md`), que trata dos metadados e da validade de cada item do contexto. O cenário de qualidade que confirma as duas decisões é o CQ-03 (`cq03_estoque_de_sexta_oferta_de_segunda.md`).
>
> A distribuidora, o comprador, o desconto de 8% e as preferências citadas são fictícios.

## Contexto e problema

O registro da tarefa (ADR 0001) pertence a uma tarefa. O que o agente de atendimento sabe de um cliente de uma tarefa para outra fica numa memória de longo prazo, separada por cliente. A documentação do LangGraph descreve esse tipo de memória como compartilhada entre conversas e organizada em namespaces, que costumam incluir o identificador do usuário ou da organização.

Os fatos que aparecem numa conversa são de dois tipos. Alguns têm sistema de origem e dono: o desconto do contrato e o limite de crédito estão no ERP, sob responsabilidade do comercial e do financeiro. Outros só existem na conversa: a preferência por entrega às terças, ou a exigência de que a nota fiscal traga o número do pedido de compra do cliente.

Em agosto, o comprador citou numa conversa um desconto negociado de 8%, e a memória guardou o valor. Em setembro, a renovação do contrato mudou o desconto no ERP. A memória virou uma cópia sem invalidação de um dado que tem dono, e a retomada de segunda-feira ofereceu o desconto antigo.

O que a memória do cliente deve guardar, e como uma memória muda sem perder a história?

## Fatores de decisão

- Nenhum valor que pertence a um sistema de origem fica guardado na memória, onde a invalidação da fonte não o alcança.
- As preferências que só existem na conversa precisam sobreviver de uma tarefa para outra, com procedência.
- Uma memória que muda não apaga a anterior, para que uma decisão antiga continue explicável.
- Os dados pessoais do comprador precisam poder ser eliminados ao fim do tratamento (LGPD, art. 16).
- A latência da conversa.
- O conteúdo que chega de documentos e de ferramentas não deve virar memória sem que o comprador o tenha dito.

## Opções consideradas

1. Sem memória de longo prazo: cada tarefa começa só do seu registro.
2. Perfil único por cliente, regenerado pelo modelo a cada conversa, com tudo o que o comprador disse.
3. Coleção de itens com o valor de qualquer fato citado, inclusive os que têm sistema de origem.
4. Coleção de itens com o valor só dos fatos sem sistema de origem, com procedência; endereço para os fatos com dono; substituição por acréscimo; gravação em segundo plano.

## Resultado da decisão

Opção escolhida: **4**. É a única das quatro que mantém as preferências entre tarefas sem criar uma cópia, fora do alcance da invalidação, de um dado que tem dono.

Regras que decorrem da decisão:

1. **Mapa de fatos com dono.** A arquitetura mantém a lista dos tipos de fato que têm sistema de origem, com o dono e o endereço de cada um. Na distribuidora: desconto e condição comercial (contrato no ERP, comercial), limite de crédito (ERP, financeiro), endereços de entrega (cadastro do cliente) e preço por cliente (ERP, comercial).
2. **Endereço, não valor.** Quando um fato da conversa corresponde a um tipo do mapa, a memória guarda o endereço do dado no sistema de origem. O valor é lido na fonte quando for necessário, com a validade do ADR 0002.
3. **Valor com procedência.** Um fato sem sistema de origem é guardado com o valor e com a procedência: a conversa, a data e o identificador do contato que o disse. Também leva o escopo (o cliente) e o estado (vigente ou substituído).
4. **Divergência com a fonte.** Se o comprador afirma um valor diferente do que está no sistema de origem (por exemplo, "nosso desconto é 8%"), a memória não grava o valor. O agente usa o valor da fonte, registra a divergência como evento da tarefa e a encaminha ao vendedor responsável pelo cliente.
5. **Substituição por acréscimo.** Uma memória nova não sobrescreve a antiga: entra como item novo, com o vínculo "substitui" para o item anterior. O item substituído deixa de entrar no contexto e continua no histórico, que segue a política de retenção. É o registro só de acréscimo do Event Sourcing, aplicado à memória.
6. **Gravação em segundo plano.** O gravador de memória roda ao fim de cada conversa, fora do caminho da resposta. A documentação do LangGraph aponta o custo de gravar durante a conversa (decidir o que guardar aumenta a latência) e o de gravar em segundo plano (atualizações espaçadas podem deixar outra conversa sem a informação nova). Disparar a gravação no fim de cada conversa, e não por um agendamento, reduz esse intervalo.
7. **Só o que o comprador disse.** Só vira memória o que o comprador disse na conversa. O conteúdo de documentos anexados e de resultados de ferramentas não vira memória. O tratamento de conteúdo injetado fica para as decisões do tema proteção e supervisão.
8. **Dados pessoais.** Vale a regra 11 do ADR 0001: os dados estruturados do contato ficam no cadastro de contatos, referenciados por identificador, e o texto livre da memória é cifrado com uma chave por titular.
9. **Escopo.** A memória é separada por cliente, e um item de um cliente nunca entra no contexto de outro. Quem pode ler cada item, por papel e por atributo, fica para as decisões do tema autoridade.

### Consequências

- Bom: a mudança de um dado no sistema de origem chega à próxima tarefa sem depender da memória.
- Bom: as preferências sobrevivem entre tarefas, com a conversa, a data e o contato que as originaram.
- Bom: a história da memória fica preservada, e uma decisão antiga continua explicável pelo item que estava vigente.
- Bom: a divergência entre o que o comprador diz e o que está no sistema de origem chega a uma pessoa.
- Ruim: cada fato com dono exige uma leitura na fonte quando é usado.
- Ruim: o mapa de fatos com dono precisa ser mantido junto com os sistemas de origem.
- Ruim: a classificação de um fato da conversa num tipo do mapa é feita pelo modelo e pode errar. Um fato com dono classificado como preferência viraria uma cópia; por isso a verificação abaixo procura valores de campos do mapa na memória.
- Neutro: uma preferência dita no início de uma conversa só fica disponível para outra conversa depois que a primeira termina.

### Confirmação

- **Teste de fato com dono:** numa conversa simulada, o comprador cita o desconto; verificar que a memória guarda o endereço do contrato, e não o valor.
- **Teste de divergência:** com o desconto alterado na fonte, o comprador afirma o valor antigo; verificar que o agente usa o valor da fonte, que a memória não grava o valor e que a divergência chega ao vendedor.
- **Teste de substituição:** o comprador muda a preferência de entrega; verificar que o item novo tem o vínculo "substitui", que o antigo deixa de entrar no contexto e que continua no histórico.
- **Teste de documento:** anexar um documento que afirma uma condição comercial; verificar que nada do documento vira memória.
- **Teste de escopo:** com dois clientes, verificar que nenhum item de um entra no contexto do outro.
- **Verificação periódica da memória:** procurar na memória valores de campos do mapa de fatos com dono.
- **Critério de aceite:** o cenário de qualidade CQ-03, medida 4.

Estes são procedimentos propostos. Nenhum resultado de execução está registrado aqui.

## Prós e contras das opções

### 1. Sem memória de longo prazo

- Bom: nenhuma cópia fora da fonte; nada a manter.
- Ruim: o comprador repete as preferências a cada tarefa.

### 2. Perfil único regenerado pelo modelo

- Bom: um documento só por cliente, fácil de pôr no contexto.
- Ruim: guarda valores que têm dono, fora do alcance da invalidação.
- Ruim: cada regeneração parte do perfil anterior, e a documentação do LangGraph registra que essa atualização fica mais sujeita a erro à medida que o perfil cresce.
- Ruim: a regeneração sobrescreve a versão anterior, e a história se perde.

### 3. Coleção com o valor de qualquer fato

- Bom: itens pequenos e fáceis de gravar.
- Ruim: guarda valores que têm dono, como o desconto de agosto.
- Ruim: o modelo precisa decidir o que apagar ou atualizar, e a documentação do LangGraph registra que alguns modelos tendem a inserir demais e outros a atualizar demais.

### 4. Valor só sem dono, endereço com dono, substituição por acréscimo

- Bom: cobre os fatores de decisão.
- Ruim: leituras na fonte, mapa de fatos a manter e classificação feita pelo modelo.

## Mais informações

- **Padrão e princípio de partida:** dados como produto, princípio do data mesh (Dehghani), do qual vêm o dono e o endereço único do dado. Dehghani formulou o princípio para dados analíticos, que ela define como uma visão temporal e agregada dos fatos do negócio; a memória do cliente é a parte do agente mais próxima dessa definição. O Event Sourcing, do Azure Architecture Center, é a referência para a substituição por acréscimo.
- **Tipos de memória:** a documentação do LangGraph distingue memória semântica (fatos), episódica (experiências) e procedural (instruções, código e pesos do modelo). Esta decisão trata da memória semântica do cliente. A procedural, no agente de atendimento, são as instruções versionadas do ADR 0001.
- **Fontes, consultadas em 19 e 20/10/2026:**
  - LangGraph, "Memory": https://docs.langchain.com/oss/python/langgraph/memory
  - Zhamak Dehghani, "How to Move Beyond a Monolithic Data Lake to a Distributed Data Mesh", 20/05/2019: https://martinfowler.com/articles/data-monolith-to-mesh.html
  - Zhamak Dehghani, "Data Mesh Principles and Logical Architecture", 03/12/2020: https://martinfowler.com/articles/data-mesh-principles.html
  - Microsoft, Azure Architecture Center, "Event Sourcing pattern": https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing
  - Lei nº 13.709/2018 (LGPD), art. 16: https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709compilado.htm
  - MADR 4.0.0: https://adr.github.io/madr/
