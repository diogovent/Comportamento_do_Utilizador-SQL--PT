# 👤 Comportamento_do_Utilizador-SQL--PT

> Construção de um perfil comportamental completo dos usuários de um programa de pontos baseado na plataforma **Twitch**, utilizando SQL puro (SQLite).

---

## 📋 Contexto do Projeto

Este é um programa de fidelidade em que usuários acumulam pontos através de ações de engajamento na Twitch (mensagens no chat, presença em streams, streaks) e podem resgatar esses pontos por itens de um universo RPG (espadas, armaduras, cajados, etc.).

A query constrói uma **tabela analítica one-row-per-user** pronta para segmentação, clustering, modelos de churn ou dashboards de CRM.

Os dados foram retirados deste site: https://www.kaggle.com/datasets/teocalvo/teomewhy-loyalty-system

Para poder rodar o projeto na sua máquina tem de transferir o documento "etl_projeto.sql" e se quiser ver as minhas anotações pode abrir (por aqui o git hub) o "user_behavioral_profile.sql".

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

O meu é meu projeto é organizado em **11 CTEs encadeadas**:

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


