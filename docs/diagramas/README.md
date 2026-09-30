# Diagramas

Diagramas derivados de `docs/descricao.md` e `docs/decisoes-e-ajustes.md`
(visão C4 — nível 2, containers), em Mermaid.

## Versão final

Corrigida depois do confronto com o código real (ver
`docs/comparacao-v1.md` e `docs/decisoes-e-ajustes.md`).

- `estrutural.mmd` — visão de containers.
- `sequencia.mmd` — fluxo principal, execução de um scan completo.
- `sequencia-falha.mmd` — cenário de falha: servidor de inferência fora do
  ar ou resposta inválida. Sem equivalente na v1.

## Versão inicial (v1)

Produzida só a partir da descrição em prosa, sem acesso ao código. Mantida
para permitir a comparação antes/depois.

- `estrutural-v1.mmd` — visão de containers.
- `sequencia-v1.mmd` — fluxo principal.
