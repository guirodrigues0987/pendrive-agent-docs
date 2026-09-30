# Decisões e Ajustes de Arquitetura — PendriveAgent

Este documento registra decisões de projeto identificáveis diretamente no
código-fonte do PendriveAgent, junto com o raciocínio declarado para cada
uma. O objetivo não é listar toda escolha de implementação, mas as decisões
que moldam a arquitetura e que valem a pena carregar para os diagramas e para
quem for manter ou estender o sistema no futuro.

## Decisão 1 — Coleta determinística em Python; o modelo de linguagem só interpreta e resume

**Contexto.** A primeira versão conceitual do agente seguiria um padrão comum
de agentes baseados em LLM: o modelo receberia acesso a um conjunto de
ferramentas (as mesmas três funções de coleta hoje expostas em `tools.py`
como `TOOL_SCHEMAS`) e decidiria, em tempo de execução, quais chamar e em que
ordem, via tool-calling/function-calling nativo da API de chat completions.

**Decisão.** Essa abordagem foi abandonada. No fluxo efetivamente
implementado em `agent.py`, o Python executa diretamente e sempre o mesmo
conjunto fixo de três coletas — processos, conexões de rede e itens de
inicialização — sem perguntar ao modelo o que coletar. O modelo de linguagem
entra na jogada só depois: ele recebe os dados já coletados (e o diff em
relação ao scan anterior, quando existir) formatados como texto em um único
prompt, e sua única responsabilidade é interpretar esses dados e escrever o
resumo final em português. O mecanismo de `TOOL_SCHEMAS`/`TOOL_REGISTRY`
continua presente em `tools.py`, mas não é exercitado pelo caminho principal
de execução do agente — é um resquício da abordagem anterior.

**Motivo registrado no código.** O próprio `agent.py` documenta o motivo
diretamente em sua docstring de módulo: modelos pequenos (7B–8B), rodando via
llama.cpp, apresentam bugs de parsing no tool-calling ao tentarem chamar
várias ferramentas na mesma resposta — descrito no código como um bug
conhecido de formatação nativa de chamada de função no `llama-server`. Como o
conjunto de ferramentas do scan é sempre o mesmo e sempre executado por
inteiro, não havia necessidade real de delegar ao modelo a decisão de "quais
ferramentas chamar" — decisão que, na prática, ele executaria de forma
instável.

**Consequências.** Ganha-se previsibilidade: a coleta de dados nunca falha
por erro de formatação do modelo, porque o modelo nunca participa dela. Em
contrapartida, o agente perde flexibilidade — não é possível, por exemplo,
pedir ao agente em linguagem natural para rodar só uma das três coletas, ou
adicionar uma nova coleta sem alterar `agent.py` diretamente. Essa é uma
decisão de arquitetura coerente com a restrição de rodar com modelos pequenos
e locais (ver `docs/descricao.md`, seção Restrições), mas que amarra a
extensibilidade do sistema ao código Python, e não mais à capacidade do
modelo.

---

Nenhuma outra decisão de arquitetura está documentada explicitamente no
código-fonte (por exemplo, em comentários ou docstrings) além da acima. Outras
escolhas de implementação existem — como os critérios de whitelist ou os
valores fixos de limite e timeout — mas, por não haver justificativa
registrada para elas no código, estão listadas como lacunas em
`docs/descricao.md`, e não como decisões documentadas aqui.

## Ajustes feitos nos diagramas (v1 → versão final)

Os diagramas iniciais (`estrutural-v1.mmd` e `sequencia-v1.mmd`) foram
produzidos simulando um arquiteto sem acesso ao código-fonte, só à prosa de
`docs/descricao.md`. Depois de ler o código real e registrar as divergências
em `docs/comparacao-v1.md`, os diagramas finais (`estrutural.mmd`,
`sequencia.mmd`) corrigem o que se provou errado, e um terceiro diagrama
novo (`sequencia-falha.mmd`) cobre um cenário que a v1 nunca tentou modelar.
Os arquivos v1 foram mantidos na pasta para permitir a comparação
antes/depois. A seguir, cada ajuste, o que motivou a mudança e por quê.

**Ordem de salvamento do snapshot.** A v1 do diagrama de sequência colocava
`Agent->>History: salva snapshot coletado` como o último passo de toda a
jornada, depois da chamada ao LLM e da impressão do resumo — uma suposição
razoável para quem lê a tarefa como "a jornada termina com o relatório salvo
no histórico", mas que não correspondia ao código. Em `agent.py`, o
snapshot é gravado em disco logo depois de calcular o diff e *antes* de
montar o prompt ou chamar o servidor de inferência. Isso foi corrigido em
`sequencia.mmd`: o bloco de salvar o snapshot agora aparece antes da etapa
de montagem do prompt e da chamada ao LLM, e uma nota explícita registra que
o snapshot já está em disco àquela altura, independente do que acontecer
depois. O diagrama estrutural também ganhou numeração nas setas principais
(1 a 7) para deixar essa ordem visível mesmo num diagrama de containers, que
por natureza não é sequencial.

**Momento da consulta à whitelist.** A v1 mostrava `tools.py` "conversando"
com `whitelist.json` como uma troca de mensagens em tempo de execução,
repetida a cada coleta (uma vez para processos, outra para itens de
inicialização) — um modelo plausível para quem não sabe que o carregamento
é feito por uma variável de módulo. O código mostra que a leitura do arquivo
acontece uma única vez, no momento em que `tools.py` é importado, e que as
checagens de "é conhecido?" feitas depois são só consultas a um conjunto já
carregado em memória, sem nenhuma nova mensagem trocada com o arquivo. Em
`sequencia.mmd`, isso foi corrigido: agora há uma única interação com
`whitelist.json`, logo após a importação do módulo e antes de qualquer
coleta, seguida de uma nota explicando que os nomes ficam em memória; as
duas checagens que antes apareciam como mensagens para `Whitelist` viraram
automensagens de `Tools` para si mesmo, representando o teste de
pertencimento em memória.

**Novo diagrama: cenário de falha do servidor de inferência.** A v1 não
tinha nenhum equivalente, porque a instrução original pedia só o fluxo
principal. `sequencia-falha.mmd` é inteiramente novo e cobre os dois
cenários pedidos, modelados exatamente como o código real os trata (ou não
trata): quando o servidor está fora do ar ou responde com erro HTTP, o
código captura a exceção, imprime uma mensagem de erro e encerra o processo
de forma controlada (`sys.exit(1)`) — isso está representado em um ramo
`alt`. Já quando o servidor responde com HTTP 200 mas com corpo vazio, JSON
inválido, ou uma estrutura sem `choices`/`message`/`content`, o código não
trata nada disso: a exceção correspondente propaga sem captura e o processo
encerra com um traceback bruto. Esse segundo ramo do diagrama registra
explicitamente, em uma nota, que se trata de uma lacuna do código real, e
não de uma omissão do diagrama — é o próprio sistema que não tem esse
tratamento.

**Diagrama estrutural: rótulo da whitelist e numeração de ordem.** Além da
numeração de passos já mencionada, o rótulo da aresta entre `tools.py` e
`whitelist.json` foi reescrito de "lê nomes conhecidos" (que sugeria uma
leitura recorrente) para "carrega uma única vez, na importação do módulo" —
incorporando diretamente, no diagrama de containers, um fato que só tinha
sido registrado em prosa em `docs/descricao.md`.

Nenhum container ou módulo foi removido ou adicionado nos diagramas finais
em relação à v1: as correções foram todas de ordem, de granularidade das
interações (mensagem trocada vs. consulta em memória) e de cobertura
(adição do cenário de falha) — não de existência de componentes.
