# Comportamento_do_Utilizador-SQL--PT


Claro! Aqui está o texto raw para colares diretamente no editor do GitHub (ficheiro README.md):
markdown# 👤 User Behavioral Profile — SQL Analysis

> Construção de um perfil comportamental completo dos usuários de um programa de pontos baseado na plataforma **Twitch**, utilizando SQL puro (SQLite).

---

## 📋 Contexto do Projeto

Este é um programa de fidelidade em que usuários acumulam pontos através de ações de engajamento na Twitch (mensagens no chat, presença em streams, streaks) e podem resgatar esses pontos por itens de um universo RPG (espadas, armaduras, cajados, etc.).

A query constrói uma **tabela analítica one-row-per-user** pronta para segmentação, clustering, modelos de churn ou dashboards de CRM.

---

## 🗃️ Base de Dados

| Tabela | Linhas | Descrição |
|---|---|---|
| `clientes` | 4.962 | Cadastro de usuários, saldo de pontos e flags de plataforma |
| `transacoes` | 293.615 | Movimentos de pontos (crédito e débito) |
| `transacao_produto` | 293.897 | Ligação entre transações e produtos |
| `produtos` | 118 | Catálogo de ações de engajamento e itens RPG |
| `clientes_d28` | 160 | Tabela auxiliar de atividade recente |
| `relatorio_diario` | 618 | Relatório acumulado de transações por dia |

### Origens das transações (`DescSistemaOrigem`)
| Plataforma | Transações | % |
|---|---|---|
| Twitch | 293.171 | 99,8% |
| Cursos | 444 | 0,2% |

### Produtos mais frequentes
| Produto | Categoria | Transações |
|---|---|---|
| ChatMessage | chat | 242.010 |
| Lista de presença | present | 40.350 |
| Presença Streak | present | 2.989 |
| Resgatar Ponei | ponei | 1.900 |
| Churn_5pp / Churn_2pp / Churn_10pp | churn_model | 3.706 |

> Os itens RPG (espadas, armaduras, etc.) aparecem apenas em transações de **resgate**, onde o cliente **gasta** pontos acumulados.

---

## 📦 Métricas Geradas

| Categoria | Métrica | Janelas |
|---|---|---|
| 🔁 Transações | Quantidade de transações | Vida, D7, D14, D28, D56 |
| ⏱️ Recência | Dias desde a última transação | — |
| 🗓️ Antiguidade | Dias desde o cadastro | — |
| 🛒 Produto | Produto mais utilizado | Vida, D7, D14, D28, D56 |
| 🏷️ Categoria | Categoria do produto mais utilizado | Vida, D7, D14, D28, D56 |
| 💰 Saldo | Saldo atual de pontos | — |
| ➕ Créditos | Pontos recebidos | Vida, D7, D14, D28, D56 |
| ➖ Débitos | Pontos gastos (resgates) | Vida, D7, D14, D28, D56 |
| 📅 Dia da semana | Dia com mais transações | D28 |
| 🌓 Período do dia | Manhã / Tarde / Noite / Madrugada | D28 |
| 📊 Engajamento | Proporção transações D28 / Vida | — |
| 🔗 Plataformas | Twitch, YouTube, Email, BlueSky, Instagram conectados | — |

> **Janelas temporais:** `Vida` = histórico completo · `D7/D14/D28/D56` = últimos 7, 14, 28 ou 56 dias.

---

## 🗂️ Arquitetura da Query

O script é organizado em **11 CTEs encadeadas**:
transacoes             clientes         transacao_produto   produtos
│                    │                     │               │
▼                    ▼                     └──────┬────────┘
tb_transações         tb_cliente                       ▼
│                    │               tb_transação_produto
│                    │                       │
▼                    │               tb_cliente_produto
tb_sumário_transações     │                       │
│                    │               tb_cliente_produto_rn
│                    │                       │
tb_cliente_dia            │               ┌───────┘
│                    │               │
tb_cliente_dia_rn         │               │
│                    │               │
tb_cliente_periodo        │               │
│                    │               │
tb_cliente_periodo_rn     │               │
│                    │               │
└────────────────────┴───────────────┘
│
tb_join
│
SELECT final
(+ Engajamento_D28_Vida)

### Descrição de cada CTE

| # | CTE | Responsabilidade |
|---|---|---|
| 1 | `tb_transações` | Parse de datas, cálculo de `Diff_Date` (dias decorridos) e hora da transação |
| 2 | `tb_cliente` | Idade na base + flags de plataformas conectadas |
| 3 | `tb_sumário_transações` | Contagem de transações, saldo e pontos por janelas temporais |
| 4 | `tb_transação_produto` | Enriquece transações com nome e categoria do produto |
| 5 | `tb_cliente_produto` | Frequência de uso por produto, cliente e janela |
| 6 | `tb_cliente_produto_rn` | `ROW_NUMBER` para produto mais usado em cada janela |
| 7 | `tb_cliente_dia` | Transações por dia da semana (D28) |
| 8 | `tb_cliente_dia_rn` | `ROW_NUMBER` para dia mais ativo |
| 9 | `tb_cliente_periodo` | Transações por período do dia (D28) |
| 10 | `tb_cliente_periodo_rn` | `ROW_NUMBER` para período mais ativo |
| 11 | `tb_join` | Consolidação de todas as CTEs em uma linha por cliente |

---

## ⚠️ Cobertura do Output

A base tem **4.962 clientes**, mas **1.469 (~30%) nunca realizaram uma transação**. Como o output parte de `tb_sumário_transações` (que só produz linhas para clientes com ao menos uma transação), esses clientes inativos são **excluídos do resultado final**.

Para incluir todos os clientes (ativos e inativos), substituir `tb_sumário_transações` como base e usar `clientes` com LEFT JOINs.

---

## 🗃️ Schema das Tabelas de Origem

```sql
transacoes
├── IdTransacao          (PK)
├── IdCliente            (FK → clientes)
├── QtdePontos           (positivo = crédito | negativo = resgate)
├── DtCriacao
└── DescSistemaOrigem    ('twitch' | 'cursos')

clientes
├── idCliente            (PK)
├── qtdePontos           (saldo atual — equivale a SUM(transacoes.QtdePontos))
├── DtCriacao
├── DtAtualizacao
├── flTwitch             (1 = canal conectado)
├── flYouTube
├── flEmail
├── flBlueSky
└── flInstagram

transacao_produto
├── idTransacaoProduto   (PK)
├── IdTransacao          (FK → transacoes)
├── IdProduto            (FK → produtos)
├── QtdeProduto
└── vlProduto

produtos
├── IdProduto            (PK)
├── DescNomeProduto
├── DescDescricaoProduto
└── DescCategoriaProduto
```
## 💡 Decisões Técnicas

- **`julianday()`** — cálculo de diferença de datas em dias no SQLite, sem depender de extensões externas.
- **`substr(DtCriacao, 1, 19)`** — remove milissegundos (`.114000`) e fusos horários que existem na coluna `DtCriacao`, garantindo compatibilidade com `datetime()` e `strftime()`.
- **`ROW_NUMBER()` com desempate** — o critério secundário `ORDER BY ... DESC, DescNomeProduto ASC` garante resultados determinísticos em caso de empate entre dois produtos com a mesma frequência.
- **`NULLIF` no engajamento** — `1.0 * D28 / NULLIF(Vida, 0)` evita erro de divisão por zero para clientes sem histórico.
- **`COALESCE` consistente** — `Dia_Semana` usa `'N/A'` (texto) e não `-1` (integer), porque `strftime('%w', ...)` retorna TEXT no SQLite — misturar tipos pode causar comportamentos inesperados em ferramentas analíticas.
- **Pontos negativos como valores negativos** — facilita o cálculo direto de saldo e a distinção entre crédito/débito sem precisar de colunas extras.
- **Categoria do produto incluída** — além do nome do produto mais usado, a query retorna a categoria (`chat`, `present`, `espada`, `armadura`, etc.), útil para segmentações de alto nível.







📋 Objetivo

Este projeto constrói uma tabela analítica que consolida o perfil comportamental de cada usuário, combinando histórico de transações,
padrões temporais e uso de produtos. 

O resultado é uma base pronta para segmentação, modelos de machine learning ou dashboards de CRM.

---

Como aceder

  Este projeto foi feito no Visual Studio Code com a extenção do SQL Lite, o link dos dados retirados (https://www.kaggle.com/datasets/teocalvo/teomewhy-loyalty-system)

  No repusitório tem uns quanto ficheiros sendo o "etl_projeto.sql" é o projeto final em que se puser no SQL Lite irá funcionar perfeitamente.

  O ficheiro de nome "user_behavioral_profile.sql" tem os meus comentários para ajudar entender o meu projeto.



---

Metas

 - Quantidade de transações históricas (vida, D7, D14, D28, D56);
 - Dias desde a última transação;
 - Idade na base (quando a pessoa se cadastrou);
 - Produto mais usado (vida, D7, D14, D28, D56);
 - Saldo de pontos atual;
 - Pontos acumulados positivos (vida, D7, D14, D28, D56);
 - Pontos acumulados negativos (vida, D7, D14, D28, D56);
 - Dias da semana mais ativos (D28);
 - Período do dia mais ativo (D28);
 - Engajamento em D28 versus Vida.

Ps: "D" - Reference ao nº de dias (ex.: D28 = vigésimo oitavo dia)

---

🗂️ Estrutura do Código
O script é organizado em CTEs (Common Table Expressions) encadeadas, seguindo o fluxo abaixo:

## Arquitetura do Projeto

```mermaid
flowchart TD
    A[transacoes] --> B[tb_transacoes]
    C[clientes] --> D[tb_cliente]
    A --> E[transacao_produto]
    F[produtos] --> E

    B --> G[tb_sumario_transacoes]
    B --> H[tb_cliente_dia]
    H --> I[tb_cliente_dia_rn]
    B --> J[tb_cliente_periodo]
    J --> K[tb_cliente_periodo_rn]

    E --> L[tb_transacao_produto]
    L --> M[tb_cliente_produto]
    M --> N[tb_cliente_produto_rn]

    G --> O[tb_join]
    D --> O
    N --> O
    I --> O
    K --> O

    O --> P[SELECT final]
    P --> Q[Engajamento_28_Vida]
````

---

Descrição de cada CTE

| #  | CTE                         | Responsabilidade |
|----|-----------------------------|------------------|
| 1  | tb_transacoes              | Parse de datas, cálculo de `Diff_Date` (dias decorridos) e hora da transação |
| 2  | tb_cliente                 | Idade na base do cliente |
| 3  | tb_sumario_transacoes      | Contagem de transações, saldo e pontos por janelas temporais |
| 4  | tb_transacao_produto       | Enriquecimento com nome e categoria do produto |
| 5  | tb_cliente_produto         | Frequência de uso por produto, cliente e janela |
| 6  | tb_cliente_produto_rn      | Identificação do produto mais usado com `ROW_NUMBER()` |
| 7  | tb_cliente_dia             | Transações por dia da semana (D28) |
| 8  | tb_cliente_dia_rn          | Dia da semana mais ativo |
| 9  | tb_cliente_periodo         | Transações por período do dia (D28) |
| 10 | tb_cliente_periodo_rn      | Período do dia mais ativo |
| 11 | tb_join                    | Consolidação final: uma linha por cliente |

---

⚠️ Cobertura do Output
A base tem 4.962 clientes, mas 1.469 (~30%) nunca realizaram uma transação. Como o output parte de tb_sumário_transações 
(que só produz linhas para clientes com ao menos uma transação), esses clientes inativos são excluídos do resultado final.
Para incluir todos os clientes (ativos e inativos), substituir tb_sumário_transações como base e usar clientes com LEFT JOINs.

---

🗃️ Schema das Tabelas de Origem

## Modelo de Dados

---

💡 Decisões Técnicas

julianday() — cálculo de diferença de datas em dias no SQLite, sem depender de extensões externas.

substr(DtCriacao, 1, 19) — remove milissegundos (.114000) e fusos horários que existem na coluna DtCriacao, garantindo compatibilidade com datetime() e strftime().

ROW_NUMBER() com desempate — o critério secundário ORDER BY ... DESC, DescNomeProduto ASC garante resultados determinísticos em caso de empate 
entre dois produtos com a mesma frequência.

NULLIF no engajamento — 1.0 * D28 / NULLIF(Vida, 0) evita erro de divisão por zero para clientes sem histórico.
COALESCE consistente — Dia_Semana usa 'N/A' (texto) e não -1 (integer), porque strftime('%w', ...) retorna TEXT no SQLite 
— misturar tipos pode causar comportamentos inesperados em ferramentas analíticas.

Pontos negativos como valores negativos — facilita o cálculo direto de saldo e a distinção entre crédito/débito sem precisar de colunas extras.

Categoria do produto incluída — além do nome do produto mais usado, a query retorna a categoria (chat, present, espada, armadura, etc.), 
útil para segmentações de alto nível.

