# MVP — Machine Learning & Analytics (Data Engineering / Databricks)
**Autor:** Gustavo Alves  
**Data:** 21/12/2025  
**Matrícula:** 4052025001911  
**Dataset:** Chess Game Dataset (Kaggle) — Chess.com user games (~60k)  
Link: [Kaggle](https://www.kaggle.com/datasets/adityajha1504/chesscom-user-games-60000-games)

---

## 1) Contexto e objetivo
Este projeto implementa um pipeline de dados em **Databricks** usando a arquitetura **Medallion (Bronze &rarr; Silver &rarr; Gold)** para ingerir, padronizar e analisar partidas de xadrez do Chess.com.

O objetivo é produzir **tabelas analíticas (Gold)** que respondem perguntas de negócio/insights sobre as partidas, distribuição de resultados, ou ainda se existe vantagem em jogar com as brancas ou com as pretas, etc.

---

## 2) Perguntas respondidas (Q1–Q8)
1. Quantas partidas foram registradas ao longo do tempo? (mensal/anual)  
2. Qual a distribuição de resultados (vitória/empate/derrota) no geral e por `time_class`?  
3. Existe vantagem estatística para as brancas? (taxa de vitória sem empates)  
4. Como `white_rating - black_rating` influencia o resultado?  
5. Como os tipos de término variam por controle de tempo?  
6. Quais aberturas (ECO) são mais frequentes e como variam por `time_class`?  
7. Quais ECOs “performam melhor” (win rate), aplicando mínimo de amostra (N≥50)?  
8. Quais jogadores mais participam e como performam (volume + win rate), por modalidade?

---

## 3) Estrutura do repositório (hierarquia)

A organização segue o padrão **Medallion (Bronze &rarr; Silver &rarr; Gold)** e separa claramente:
- **código executável** (notebooks)
- **evidências com outputs renderizados** (para avaliação sem Databricks)

### Árvore de pastas
``` text
mvp_sprint_3_data_engineering
├── .databricks/
|   └── commit_outputs
├── notebooks/
│   ├── 00_setup.ipynb
|   ├── 01_bronze_ingestion.ipynb
│   ├── 02_silver_transformations.ipynb
│   └── 03_gold_analytics.ipynb
├── notebooks_executed_evidence/
|   ├── 00_setup_executed_evidence.ipynb
│   ├── 01_bronze_ingestion__executed.ipynb
│   ├── 02_silver_transformations__executed.ipynb
│   └── 03_gold_analytics__executed.ipynb
├── LICENSE
└── README.md
```

### O que existe em cada pasta

- `.databricks/`
  Contém arquivos de configuração do Databricks Repos (não faz parte do pipeline).
  Neste projeto, o arquivo `commit_outputs` está configurado para incluir os outputs no Git apenas 
  para notebooks em `notebooks_executed_evidence/`, garantindo que as evidências (tabelas/gráficos) fiquem versionadas sem poluir os notebooks fonte.

- `notebooks/`
  Contém os notebooks **fonte** do pipeline (para execução no Databricks), separados por camada:
  - **Bronze**: ingestão do CSV e criação da tabela raw/landing
  - **Silver**: padronização/limpeza e criação da tabela curada `workspace.default.silver_games`
  - **Gold**: agregações analíticas (tabelas *gold*) + gráficos e análises por pergunta

- `notebooks_executed_evidence/`  
  Contém cópias dos notebooks com **outputs já renderizados** (tabelas e gráficos), usados como **evidência**.
  Esta é a pasta recomendada para avaliação quando **não há acesso ao Databricks**.

---

## 4) Arquitetura Medallion (visão geral)
### Bronze (Raw / Landing)
- Ingestão do CSV raw, com o mínimo de transformação necessária.
- Objetivo: manter rastreabilidade e reprocessamento fácil, se necessário.

### Silver (Curated / Clean)
- Limpeza e padronizações:
  - tipagem de colunas (rating numérico, datas, boolean etc.);
  - normalização de resultado em **`outcome`**: `white_win`, `black_win` e `draw`;
  - criação de campos derivados:
    - `white_score` e `black_score` (win=1, draw=0.5, loss=0)
    - `rating_diff = white_rating - black_rating`
  - extração e normalização de metadados do PGN quando aplicável (ex.: ECO e Termination).

### Gold (Analytics / Serving)
- Tabelas agregadas e prontas para consumo, seja em dashboards, análises ou relatórios.
- A Gold é gerada pelo notebook **`03_gold_analytics`** e depende da tabela Silver:
  - **`workspace.default.silver_games`**

---

## 5) Como avaliar / executar (reprodutibilidade)

Este projeto pode ser consumido de **duas formas**, dependendo do seu nível de acesso ao Databricks:

### Opção A — Executar o pipeline (reprodutível)
> Requer: acesso ao Databricks + cluster ativo + permissões de leitura/escrita no schema `workspace.default`.

No Databricks, realize a execução **em ordem (Bronze &rarr; Silver &rarr; Gold)** dentro da pasta `notebooks`:
- 00_setup.ipynb
- 01_bronze_ingestion.ipynb
- 02_silver_transformations.ipynb
- 03_gold_analytics.ipynb

### Opção B — Avaliar apenas pelos outputs (sem execução)
> Esta é a opção esperada para os professores: **não é necessário acesso ao Databricks**.

No GitHub, abra os notebooks em `notebooks_executed_evidence/` (arquivos `*_executed.ipynb`). Os outputs (tabelas e gráficos) já estão renderizados.

---

## 6) Tabelas Gold geradas (outputs)
As tabelas abaixo são criadas pelo notebook **`03_gold_analytics`**:

### Q1 — Volume ao longo do tempo
- **`workspace.default.gold_games_by_month`**
  - `year_month`, `n_games`
- **`workspace.default.gold_games_by_year`**
  - `year`, `n_games`

### Q2 — Distribuição de resultados
- **`workspace.default.gold_outcomes_overall`**
  - `outcome`, `n_games`, `pct`
- **`workspace.default.gold_outcomes_by_time_class`**
  - `time_class`, `outcome`, `n_games`, `pct`

### Q3 — Vantagem das brancas (sem empates)
- **`workspace.default.gold_white_advantage`**
  - `white_wins`, `black_wins`, `draws`, `n_games`, `decided_games`,
  - `white_win_rate_no_draw`, `black_win_rate_no_draw`
- **`workspace.default.gold_white_advantage_by_time_class`**
  - métricas equivalentes segmentadas por `time_class` (+ `draw_rate`)

### Q4 — Resultado por faixa de diferença de rating (bins de 200)
- **`workspace.default.gold_outcome_by_rating_diff_bin`**
  - `bin_start`, `bin_end`, `outcome`, `n_games`, `pct_in_bin`, `n_bin_total`

### Q5 — Tipos de término por modalidade
- **`workspace.default.gold_termination_by_time_class`**
  - `time_class`, `termination_type`, `n_games`, `pct_in_time_class`

### Q6 — Frequência de ECO
- **`workspace.default.gold_eco_overall`**
  - `eco`, `n_games`, `pct`
- **`workspace.default.gold_eco_by_time_class_top15`**
  - `time_class`, `eco`, `n_games`, `pct_in_time_class` (Top 15 por modalidade)

### Q7 — Performance de ECO (com mínimo de amostra)
- **`workspace.default.gold_eco_performance_min50`**
  - `eco`, `n_games`,
  - `white_win_rate`, `draw_rate`, `black_win_rate`,
  - `white_score_avg`
  - filtro: somente ECO válido `^[A-E][0-9]{2}$` e `n_games >= 50`

### Q8 — Participação e performance de jogadores (por modalidade)
- **`workspace.default.gold_player_stats_by_time_class`**
  - `time_class`, `player`, `n_games`, `win_rate`, `avg_score`
  - construído via UNION ALL (jogador como White + jogador como Black)

---

## 7) Dicionário de dados 

### Colunas originais

- **white_username** — nome de usuário das brancas
- **black_username** — nome de usuário das pretas
- **white_id / black_id** — link/perfil do jogador
- **white_rating / black_rating** — ELO antes da partida
- **white_result / black_result** — resultado do ponto de vista de cada cor
- **time_class** — bullet, blitz, rapid, daily
- **time_control** — base+incremento (ex.: 180+2) ou formato de correspondência (daily)
- **rules** — variante (ex.: chess, chess960 etc.)
- **rated** — ranqueada (True/1) vs casual (False/0)
- **fen** — posição em Forsyth-Edwards Notation
- **pgn** — PGN (metadados + lances)

### Valores de resultado (white_result / black_result)

- win, checkmated, resigned, timeout, abandoned
- empates típicos: stalemate, agreed, repetition, 50move, insufficient, timevsinsufficient
- variantes podem aparecer (ex.: threecheck, kingofthehill)

### Controle de tempo (time_class)

- bullet — muito rápido (tipicamente < 3 min por lado)
- blitz — rápido (3–10 min por lado)
- rapid — 10–30 min por lado
- daily — correspondência (dias por lance)
Referência (Chess.com): [Time Controls](https://www.chess.com/terms/chess-time-controls)

---

## 8) Observações importantes e limitações do dataset - Disclaimer

O dataset é não-oficial e pode estar concentrado em determinados períodos/usuários.

Pode haver:
- ECO ausente (UNKNOWN) ou inconsistente no PGN;
- viés de amostra (jogadores muito ativos dominando o volume);
- “extremos” estatísticos em grupos com baixa contagem (especialmente ao segmentar muito).

As conclusões devem ser interpretadas como insights da amostra, não como retrato histórico completo do Chess.com.

---

## 9) Licença e uso

O dataset é distribuído via Kaggle: verifique termos/licença do dataset na página oficial.

Este repositório foca em fins acadêmicos/didáticos (pipeline + análises).
