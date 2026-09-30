# Descrição Arquitetural — PendriveAgent

## Escopo

O PendriveAgent é um agente de triagem de segurança que roda inteiramente em
uma máquina local, a partir de um pendrive, sem depender de conectividade com
a internet ou de qualquer serviço em nuvem. Seu propósito é coletar um
retrato do estado corrente de um computador — processos em execução, conexões
de rede ativas e itens configurados para iniciar automaticamente com o
sistema — e produzir, com a ajuda de um modelo de linguagem local, um resumo
em português voltado a uma pessoa não técnica, apontando o que parece fora do
padrão. O próprio sistema se declara explicitamente como uma camada de
leitura e triagem, não como um antivírus: ele não reconhece assinaturas de
malware conhecidas e não tem base de dados de reputação, apenas compara o
estado atual com o estado de execuções anteriores e com uma lista de itens
considerados esperados naquele ambiente. O escopo desta documentação cobre a
arquitetura do agente tal como implementada — o orquestrador em Python, os
módulos de coleta e de histórico, o arquivo de configuração de whitelist e a
integração com o servidor de inferência local — sem entrar em código-fonte
linha a linha, já que o objetivo aqui é a visão de containers e dos limites
entre eles.

## Nível da visão adotada (C4 — Nível 2, Containers)

Esta descrição e os diagramas derivados dela adotam o nível 2 do modelo C4,
ou seja, o nível de **containers**: unidades de execução ou de
armazenamento que compõem o sistema e como elas se comunicam entre si e com
o mundo externo. Não se detalha aqui a estrutura interna de classes ou
funções de cada módulo (isso seria o nível 3, de componentes), tampouco se
descrevem outros sistemas externos ao PendriveAgent além dos estritamente
necessários para entender suas fronteiras — o próprio sistema operacional do
host e o servidor de inferência local. Os containers identificados no
sistema são: o processo do agente em Python (que concentra orquestração,
coleta e construção de prompt), o arquivo de configuração de whitelist, o
diretório de histórico de scans em disco, e o servidor de modelo de
linguagem local, tratado como um container externo com o qual o agente troca
requisições HTTP.

## Limites e responsabilidades de cada módulo

O sistema é dividido em três módulos Python com responsabilidades bem
delimitadas, mais um arquivo de configuração que funciona como dado externo
lido em tempo de execução.

**`agent.py`** é o ponto de entrada e o orquestrador. Ele lê os argumentos de
linha de comando (endereço do servidor LLM e nome do modelo), dispara as
três coletas de dados do sistema, solicita ao módulo de histórico a
comparação com o scan anterior, **salva o novo snapshot coletado**, monta o
prompt textual que será enviado ao modelo de linguagem, só então faz a
chamada HTTP para o servidor de inferência e imprime o resumo retornado. A
ordem importa: o snapshot é gravado em disco *antes* de qualquer tentativa
de contato com o modelo de linguagem, então mesmo que a chamada ao LLM
falhe ou o servidor esteja fora do ar, o histórico de scans já reflete a
coleta feita naquela execução. Sua responsabilidade termina na orquestração
e na comunicação com o LLM — ele não sabe *como* os dados do sistema são
coletados nem *como* os snapshots são persistidos, apenas consome as
interfaces de `tools.py` e `history.py`. O payload enviado ao servidor de
inferência carrega duas mensagens — um prompt de sistema fixo, com as regras
de como o modelo deve se comportar, e um prompt de usuário com os dados
coletados — dentro de um único corpo JSON, com temperatura fixa em 0.2.

**`tools.py`** é o módulo responsável por toda a coleta de dados do sistema
operacional, e é o único ponto do código que precisa lidar com as diferenças
entre Windows e Linux. Ele expõe três funções de coleta — processos em
execução, conexões de rede ativas e itens de inicialização automática — cada
uma retornando uma lista de dicionários em um formato simples e consistente,
pensado para ser fácil de interpretar tanto por código quanto pelo modelo de
linguagem. Este módulo também é responsável por carregar o arquivo de
whitelist e marcar, em cada item coletado, se ele é reconhecido como
esperado naquele ambiente — mas essa carga acontece **uma única vez**, no
momento em que o módulo é importado (isto é, antes mesmo de a primeira
coleta começar), e fica retida em memória pelo resto da execução: cada
verificação de "é whitelisted?" feita durante uma coleta é apenas uma
consulta a um conjunto em memória, não uma nova leitura do arquivo. O módulo
mantém ainda, por compatibilidade com um estilo de function-calling, um
registro (`TOOL_REGISTRY`) e os esquemas (`TOOL_SCHEMAS`) dessas três
funções — resquício de uma arquitetura em que o próprio modelo decidiria
quais ferramentas invocar, hoje não exercitado pelo fluxo principal do
agente (ver `docs/decisoes-e-ajustes.md`).

**`history.py`** é responsável exclusivamente pela persistência e comparação
de snapshots de scans. Ele grava cada execução como um arquivo JSON
timestampado dentro de um diretório de scans, carrega o snapshot mais recente
anterior quando existir, e calcula a diferença entre o estado atual e esse
snapshot anterior — quais processos, conexões remotas e itens de
inicialização são novos em relação à última execução. Esse módulo não coleta
dados por conta própria nem decide o que fazer com o resultado da
comparação; ele apenas entrega ao chamador (`agent.py`) uma estrutura com o
que mudou, deixando a interpretação e a redação do resumo inteiramente a
cargo do modelo de linguagem.

**`whitelist.json`** não é código, mas funciona como um container de
configuração externo ao processo do agente. É um arquivo JSON com duas
listas de nomes (em minúsculas) — uma de nomes de processos e outra de
termos associados a itens de inicialização — considerados normais ou
esperados naquele ambiente específico. O formato é intencionalmente simples:
strings de nomes, sem metadados adicionais, editável livremente por quem usa
o agente para adaptar ao próprio ambiente. Esta documentação descreve apenas
esse formato; nenhum conteúdo real do arquivo é reproduzido aqui, por
constituir dado específico de uma máquina.

## Integrações

**llama.cpp / llama-server (ou Ollama).** O agente se comunica com um
servidor de inferência local através de uma API HTTP compatível com o
formato de chat completions da OpenAI, no endpoint `/v1/chat/completions`.
Essa compatibilidade de protocolo é o que permite ao mesmo código de
`agent.py` funcionar tanto com um `llama-server` do próprio llama.cpp (o
binário é distribuído junto com o pendrive) quanto com uma instância local do
Ollama — bastando trocar o host e o nome do modelo por parâmetro de linha de
comando. O agente não gerencia o ciclo de vida desse servidor: ele assume que
o servidor já está no ar antes de ser executado, e trata falhas de conexão
ou erros HTTP encerrando a execução com uma mensagem de diagnóstico.

**Sistema operacional do host.** A coleta de processos e conexões de rede é
feita via `psutil`, de forma cross-platform. Já a coleta de itens de
inicialização automática é específica por sistema operacional: no Windows,
o agente lê as chaves de registro `Run` em `HKEY_CURRENT_USER` e
`HKEY_LOCAL_MACHINE` via `winreg`; no Linux, ele consulta unidades systemd
habilitadas, o crontab do usuário corrente e os arquivos `.desktop` de
autostart. Em ambos os casos, a visibilidade completa de processos e conexões
de outros usuários depende de o agente ser executado com privilégios
administrativos.

**Sistema de arquivos.** Além do arquivo de whitelist, o agente depende do
sistema de arquivos local para dois fins: ler o binário do modelo de
linguagem e o executável do servidor de inferência (organizados em pastas
dentro do próprio pendrive) e gravar/ler o histórico de snapshots de scans
em um diretório dedicado. Esse diretório de scans é tratado como dado local
da máquina onde o agente roda, não como algo a ser versionado.

## Restrições

O sistema foi desenhado sob um conjunto claro de restrições. Tudo deve rodar
**100% localmente**: não há chamadas a serviços em nuvem, nem para o modelo
de linguagem, nem para qualquer outra funcionalidade — a única rede que o
agente utiliza é a comunicação HTTP local (loopback) com o servidor de
inferência rodando na mesma máquina. O sistema é pensado para **rodar a
partir de um pendrive**, carregando consigo o interpretador Python (opcional,
caso a máquina de destino não tenha um), os binários do motor de inferência
para Windows e Linux, e o próprio arquivo do modelo, de forma que baste
conectar o pendrive a um computador para executar o agente sem instalação
prévia. Por rodar em hardware variado e sem GPU dedicada na maioria dos
casos, o sistema é pensado para **modelos pequenos, quantizados** (na ordem
de 7B a 13B parâmetros) — o que, como registrado em
`docs/decisoes-e-ajustes.md`, tem impacto direto em uma decisão de
arquitetura importante do agente.

## Lacunas

Nem tudo está decidido ou definido no código-fonte lido. As lacunas a seguir
são pontos genuinamente em aberto, não inferências ou sugestões:

O código não possui testes automatizados — nenhum arquivo de teste foi
encontrado junto aos três módulos. Não há também nenhum mecanismo de
logging estruturado; toda a saída do agente é feita via `print`, o que
significa que não há um formato de log persistente para auditoria além dos
próprios snapshots salvos em `scans/`.

O crescimento do diretório de scans não é gerenciado: cada execução grava um
novo arquivo JSON e não existe rotina de expurgo ou rotação dos snapshots
antigos. Da mesma forma, o formato desses arquivos JSON não tem número de
versão nem esquema declarado — uma mudança futura na estrutura dos dados de
coleta quebraria silenciosamente a comparação com snapshots salvos por uma
versão anterior do agente.

A cobertura de itens de inicialização automática é assimétrica entre os
sistemas operacionais suportados: no Windows, apenas as chaves de registro
`Run` de usuário e de máquina são verificadas (não há checagem de
Agendador de Tarefas, de `RunOnce`, nem de serviços do Windows); no Linux, a
cobertura inclui systemd, crontab e autostart de aplicações, mas não há
suporte a macOS em nenhuma parte do código.

A lógica de whitelist usa dois critérios de comparação diferentes entre si,
sem que o código explique a diferença como intencional: para processos, a
verificação é por igualdade exata do nome (em minúsculas); para itens de
inicialização, a verificação é por correspondência de substring. Se essa
assimetria é proposital ou um detalhe de implementação não revisado não está
declarado em nenhum comentário ou commit disponível.

O tratamento de erro da chamada ao servidor de inferência é parcial, e a
fronteira entre o que é tratado e o que não é fica só visível ao ler o
código com atenção (ver `docs/diagramas/sequencia-falha.mmd`). Falha de
conexão (servidor fora do ar) e respostas HTTP de erro são capturadas
explicitamente e resultam em uma mensagem de erro amigável seguida de
encerramento controlado do processo (`sys.exit(1)`) — mas isso, mesmo assim,
sem retry, backoff ou modo degradado. Já uma resposta HTTP 200 com corpo
vazio, JSON inválido, ou um JSON válido porém sem a estrutura esperada (sem
as chaves `choices`/`message`/`content`) **não tem nenhum tratamento**: o
código deixa a exceção correspondente (erro de decodificação de JSON, ou
erro de chave/índice ausente) propagar sem captura, encerrando o processo
com um traceback bruto em vez de uma mensagem compreensível. Também não há,
no código, nenhuma forma de autenticação ou verificação de que o servidor
respondendo em `--host` é de fato o servidor de inferência esperado — a
integração assume implicitamente que o ambiente local é confiável.

Por fim, parâmetros como o limite de processos exibidos (30), o tamanho
máximo de cada seção do prompt em caracteres (6000) e o timeout da chamada
ao modelo (1800 segundos) estão fixos no código como constantes, sem exposição
via linha de comando — não é possível saber, só pela leitura do código, se
essa rigidez é uma decisão deliberada de simplicidade ou um ponto ainda não
desenvolvido.
