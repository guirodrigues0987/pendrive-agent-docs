# Comparação — Diagramas v1 vs. Código Real

> **Nota.** Este documento registra o estado de `docs/descricao.md` *antes*
> dos ajustes. As omissões apontadas na seção 3 (por exemplo, a ordem do
> salvamento do snapshot e o momento de carga da whitelist) valiam para a
> descrição daquela época; `docs/descricao.md` foi atualizado depois com as
> informações que faltavam.

Este documento confronta `docs/diagramas/estrutural-v1.mmd` e
`docs/diagramas/sequencia-v1.mmd` — produzidos a partir apenas de
`docs/descricao.md`, sem acesso ao código — com o código-fonte real em
`D:\PendriveAgent` (`agent.py`, `tools.py`, `history.py`). Os diagramas **não
foram corrigidos** neste passo; o objetivo aqui é só avaliar, com prova em
código, onde a inferência feita só a partir da descrição acertou, onde
errou, e onde a própria descrição não dava informação suficiente para
decidir.

## 1. O que foi inferido corretamente (com justificativa)

**Os três módulos rodam no mesmo processo/container.** Os dois diagramas
agrupam `agent.py`, `tools.py` e `history.py` dentro de um único boundary
("Processo do Agente"). Isso está correto: `agent.py` importa as funções de
`tools.py` e `history.py` diretamente (`from tools import list_processes,
list_network_connections, list_startup_items` e `from history import
load_latest_snapshot, save_snapshot, diff_snapshots`, linhas 31–32), então
não há separação de processo ou de máquina entre eles — é tudo um único
interpretador Python em execução.

**Endpoint e verbo HTTP para o servidor de inferência.** O diagrama
estrutural e o de sequência usam `POST /v1/chat/completions`. Isso bate
exatamente com `call_llm()` em `agent.py`, que monta a requisição para
`f"{host}/v1/chat/completions"` com `method="POST"`.

**Ordem das três coletas.** O diagrama de sequência mostra `Agent` chamando
`Tools` três vezes, nesta ordem: processos, conexões de rede, itens de
inicialização. É exatamente a ordem em `run_agent()`: `list_processes(...)`
seguido de `list_network_connections()` seguido de `list_startup_items()`.

**Duas chamadas distintas a `History` para obter o snapshot anterior e
calcular o diff.** O diagrama modela isso como duas interações separadas
(`pede snapshot anterior` e depois `calcula diferença`). Isso corresponde
fielmente a `run_agent()`, que chama `load_latest_snapshot()` e, em seguida,
passa o resultado para `diff_snapshots(previous, processes, connections,
startup)` como duas chamadas de função distintas.

**Ausência de checagem de whitelist para conexões de rede.** Nenhum dos dois
diagramas conecta a coleta de conexões de rede ao container/participante
`Whitelist`. Isso está certo: `list_network_connections()` em `tools.py`
não faz nenhuma referência a `WHITELIST_PROCESSES` ou `WHITELIST_STARTUP` —
só processos e itens de inicialização são comparados com a whitelist.

**O resumo do LLM não é persistido no histórico — só os dados brutos.** O
diagrama de sequência rotula o passo final como "salva snapshot coletado",
não "salva o relatório". Isso está certo: `save_snapshot(processes,
connections, startup_items)` em `history.py` (linhas 39–52) grava só
`timestamp`, `processes`, `connections` e `startup_items` no JSON — o texto
gerado pelo LLM (`answer`) nunca é passado para essa função nem gravado em
lugar nenhum; ele só é impresso no terminal.

**`monta o prompt` como automensagem em `Agent`, não como chamada a outro
participante.** O diagrama de sequência representa a montagem do prompt
como `Agent->>Agent: monta prompt...`. Isso é coerente com o código:
`build_prompt()` é uma função definida dentro do próprio `agent.py` e
chamada localmente por `run_agent()` — não pertence a `tools.py` nem a
`history.py`.

**O cálculo do diff não toca o diretório de scans.** O diagrama de sequência
não desenha nenhuma interação entre `History` e `ScansDir` durante o passo
"calcula diferença". Isso está correto: `diff_snapshots()` em `history.py`
(linhas 67–89) opera inteiramente em memória sobre os dados já carregados
(`previous`) e os dados atuais recebidos por parâmetro — não faz nenhuma
leitura ou escrita em disco.

## 2. O que divergiu do sistema real ou foi inventado (com prova)

**Erro de sequência: o snapshot é salvo ANTES de chamar o LLM, não depois.**
O diagrama de sequência coloca `Agent->>History: salva snapshot coletado`
como o último bloco de toda a jornada, depois de `Agent->>LLM:
POST /v1/chat/completions` e depois de `Agent->>Usuario: imprime resumo do
scan`. Isso é factualmente errado. Em `agent.py`, dentro de `run_agent()`,
a ordem real é:

```python
previous = load_latest_snapshot()
diff = diff_snapshots(previous, processes, connections, startup)

snapshot_path = save_snapshot(processes, connections, startup)   # <- salva AQUI
print(f"  → snapshot salvo em: {snapshot_path}")

prompt_data = build_prompt(processes, connections, startup, diff)
...
response = call_llm(host, model, messages)                       # <- LLM chamado DEPOIS
```

(linhas ~121–138 de `agent.py`). O snapshot é gravado em disco *antes* de o
prompt ser montado e *antes* de qualquer chamada ao servidor de inferência.
Se a chamada ao LLM falhar ou travar, o snapshot já está salvo — algo que o
diagrama, ao colocar o salvamento por último, esconde completamente. Esta
não foi uma simplificação aceitável: é uma inversão da ordem real dos
efeitos colaterais do sistema.

**Erro de mecanismo: a whitelist não é consultada "ao vivo" durante a
coleta — ela é carregada uma única vez, na importação do módulo.** O
diagrama de sequência desenha `Tools->>Whitelist: consulta nomes
conhecidos` / `Whitelist-->>Tools: nomes conhecidos` como uma troca de
mensagens em tempo de execução, e a repete duas vezes (uma após coletar
processos, outra após coletar itens de inicialização), como se fosse uma
consulta pontual a cada coleta. Isso não é o que o código faz. Em
`tools.py`:

```python
WHITELIST_PROCESSES, WHITELIST_STARTUP = load_whitelist()   # linha 36, nível de módulo
```

Essa linha roda **uma única vez**, no momento em que `tools.py` é
importado — antes mesmo de `run_agent()` começar a coletar qualquer coisa.
A partir daí, `list_processes()` (linha 56: `name.lower() in
WHITELIST_PROCESSES`) e `list_startup_items()` (linha 106: `any(w in name
for w in WHITELIST_STARTUP)`) apenas testam pertencimento contra *sets* já
carregados em memória, dentro do próprio `tools.py` — não há nenhuma
segunda leitura do arquivo, nem uma "mensagem" enviada a um participante
externo chamado `Whitelist` durante a coleta. O diagrama trata como
interação de runtime, repetida, algo que na realidade é um efeito de
import-time, executado uma vez só e cacheado.

Não foram encontrados componentes puramente inventados (sem nenhuma base no
código) em nenhum dos dois diagramas — os dois erros acima são erros de
**timing/ordem de eventos**, não de existência de containers ou módulos que
não existam no sistema real.

## 3. O que a descrição não cobria e causou a suposição

**Ordem entre salvar o snapshot e chamar o LLM.** `docs/descricao.md`
descreve as responsabilidades de `agent.py` como uma lista ("dispara as
coletas, solicita a comparação, monta o prompt, chama o LLM, imprime o
resumo") que **nem sequer menciona** o passo de salvar o snapshot nessa
sequência — ele só aparece, isolado, na descrição de responsabilidades de
`history.py`. Sem uma ordem explícita, o enunciado da própria tarefa
("...até o relatório salvo no histórico") foi o que guiou a suposição de
que o salvamento seria a última etapa. Essa lacuna na descrição é a causa
direta do erro apontado na seção 2, item 1.

**Momento em que a whitelist é carregada.** `docs/descricao.md` diz apenas
que `tools.py` "é responsável por carregar o arquivo de whitelist e marcar,
em cada item coletado, se ele é reconhecido como esperado" — uma frase sobre
*responsabilidade*, não sobre *quando* o carregamento acontece (uma vez, por
execução, por item). Essa omissão é a causa direta do erro apontado na
seção 2, item 2: na ausência dessa informação, assumi (incorretamente) uma
consulta ao vivo por coleta, em vez de um carregamento único em memória.

**Existência de um ator "Usuário".** A descrição nunca nomeia um
"usuário" ou "operador" como parte do sistema — ela só menciona, de
passagem, que `agent.py` "lê os argumentos de linha de comando" e que o
resumo final é "voltado a uma pessoa não técnica". Não havendo esse
container/ator explicitamente descrito, inventei-o nos dois diagramas para
fechar a jornada ponta a ponta. O código confirma que faz sentido modelar
essa entidade (há leitura de `argparse` e impressão em `stdout`), mas ela
não é um nome retirado da descrição, e sim uma inferência para preencher a
ausência de um ponto de entrada/saída explícito no texto.

**Estrutura real da requisição ao LLM.** A descrição fala em "chamada HTTP
para o servidor de inferência" de forma genérica, sem detalhar que o corpo
da requisição carrega duas mensagens distintas (system prompt fixo +
prompt do usuário com os dados coletados) dentro do mesmo payload JSON.
Essa ausência de detalhe levou a simplificar a interação como uma única
seta genérica `POST /v1/chat/completions (prompt)`, escondendo que há, na
prática, composição de múltiplas mensagens antes do envio.

**Se o cálculo do diff toca o disco.** A descrição não afirma explicitamente
que `diff_snapshots` opera só em memória — isso teve que ser inferido (e,
neste caso, a inferência bateu com o código). Mas é importante registrar
que foi uma suposição, não uma informação dada pela descrição: o texto de
`docs/descricao.md` fala em "calcula a diferença entre o estado atual e
esse snapshot anterior" sem detalhar se isso envolve I/O adicional. O
diagrama acertou por raciocínio (a comparação não precisaria reler nada que
já não tivesse sido carregado no passo anterior), não porque a descrição
garantisse esse comportamento.
