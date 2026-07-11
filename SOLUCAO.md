# SOLUCAO.md

Status: Q0 a Q5 prontas.

## O que fiz

**Q0** — explorei os dois arquivos brutos antes de decidir qualquer correção.
Encontrei: identificador de UC em três formatos diferentes (com pontuação,
só dígitos, e com sufixo `-CANCEL`); 5 UCs que trocam de formato no meio da
série sem sobreposição de mês (mesma UC, sinalização de sistema mudou);
sufixo `-CANCEL` em 10 UCs, que guardei como flag booleana em vez de
descartar, porque o padrão de saldo crescendo sem consumo é justamente o
"crédito parado" que a Q4 pede; 7 linhas 100% duplicadas; 7 combinações
UC+mês com valores próximos mas distintos (resolvidas pela média); 4 valores
de consumo negativo (interpolados pela mediana local); uma linha de geração
sem data (descartada); e uma quebra estrutural na geração em 2024 que
documentei mas não tratei aqui — decisão de modelagem, fica para a Q3.

A função `carregar_e_limpar()` devolve os dois DataFrames limpos mais um
relatório de qualidade (etapa, registros afetados, motivo). Testei
idempotência rodando a função duas vezes e comparando o resultado.

**Q1** — implementei `ECA`, `Co` e `P` conforme a fórmula do desafio. Validei
`Σ P = 1` por mês, mas deixei explícito no notebook que essa validação prova
só a normalização algébrica, não a corretude dos `Co` individuais — quem
garante isso é a limpeza estrutural feita na Q0. Também respondi a pergunta
sobre `Σ Co = 0`: no dataset isso nunca ocorre, mas a função já trata o caso
(devolve `NaN` em vez de erro de divisão), e descrevo como trataria em
produção.

**Q2** — mesma fórmula da Q1, em SQL (DuckDB, lendo o DataFrame limpo direto,
sem exportar pra arquivo). Uso o último mês com dado real (`2026-06-01`),
já que `2026-07-01` (mês de referência do desafio) não existe na base de
consumo. Validei linha a linha contra o resultado da Q1 em Python — bate.

**Q3** — em ambos os itens, comparei um baseline (naive sazonal) com um
modelo mais elaborado, validando em jun/2026 e retreinando com jun incluído
para prever jul/2026.
- (a) Consumo das top 10 UCs: o **baseline venceu** (MAE e MAPE menores que
  Holt-Winters) — a série é curta (36 meses, ~3 ciclos anuais) para o ganho
  de modelar tendência/sazonalidade compensar a variância extra. Descobri
  durante a validação que 2 UCs de alto consumo zeram abruptamente no último
  mês sem estarem marcadas como canceladas — documentei como incerteza (pode
  ser churn não sinalizado ou atraso de leitura) e refiz a comparação com e
  sem essas UCs; a conclusão não muda.
- (b) Geração da usina: decidi treinar só com dados pós-expansão de
  capacidade (a partir de jul/2024), porque o patamar de geração mudou
  permanentemente — treinar com o regime antigo ensinaria uma capacidade que
  a usina não tem mais. Aqui o **modelo avançado venceu** (regressão linear
  com tendência + termos de Fourier, porque a série pós-quebra é curta
  demais — 23 obs — para Holt-Winters sazonal, que pede pelo menos 24).
- Pergunta de fechamento (cobertura): tratei o consumo total da carteira
  como uma série agregada própria (em vez de somar 200 previsões
  individuais, que acumularia erro). Resultado: cobertura prevista de
  **~76%** em julho/2026, déficit de ~150 mil kWh.

**Q4** — excedente = saldo acumulado das UCs canceladas (734.931 kWh,
81,6% de todo o saldo da carteira). Redistribuí com alocação proporcional
com teto ("water-filling"): prioridade = previsão da Q3 (top 10) ou consumo
médio histórico dos últimos 12 meses (demais UCs ativas); ninguém ultrapassa
1 mês do próprio consumo médio; quem já estava acima do teto não recebe
nada. Sobraram ~82 mil kWh que não couberam em ninguém (todo mundo com
espaço já atingiu o teto).

**Q5** — tabela de anomalias com tratamento de ETL sugerido para cada uma
(a maioria já apareceu na Q0/Q3). Duas ações de negócio concretas: alerta de
vacância de crédito (baseado no achado de que 5% das UCs concentram 81,6%
do crédito ocioso) e alerta de queda abrupta de consumo em UC de alto valor
sem flag de cancelamento (baseado no achado da Q3).

## O que deixei de fora (e por quê)

- Não tratei a quebra estrutural da geração em 2024 na Q0 além de
  documentar — virou decisão de janela de treino na Q3(b).
- Não recalculei saldo histórico nem tentei reconstruir o rateio de meses
  passados a partir do saldo — a Q1 pede o cálculo de `Co`/`P` a partir dos
  dados como estão, não uma reconciliação retroativa.
- Na Q3(a), não rodei um modelo por UC além do top 10 (não foi pedido) nem
  tentei um modelo multivariado (ex.: painel com efeito fixo por UC) — dado
  o tamanho da série por UC, não vi motivo para a complexidade extra.
- Não investiguei a fundo a causa raiz das 2 UCs que zeram consumo na Q3
  (churn real vs. atraso de leitura) — não tinha como decidir com o dado
  disponível; documentei a incerteza em vez de arbitrar uma resposta.
- No rebalanceamento (Q4), a prioridade usa previsão da Q3 só para as top
  10 UCs; para as demais, uso consumo médio histórico como proxy — não
  rodei um modelo de previsão para as ~176 UCs restantes, o que seria mais
  correto mas não foi pedido no escopo da Q3.
