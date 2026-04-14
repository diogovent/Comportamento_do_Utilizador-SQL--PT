# Comportamento_do_Utilizador-SQL--PT

📋 Objetivo
Este projeto constrói uma tabela analítica que consolida o perfil comportamental de cada usuário, combinando histórico de transações,
padrões temporais e uso de produtos. 
O resultado é uma base pronta para segmentação, modelos de machine learning ou dashboards de CRM.

---

Questões que tinha de chegar

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

```mermaid
erDiagram
    clientes ||--o{ transacoes : possui
    transacoes ||--o{ transacao_produto : gera
    produtos ||--o{ transacao_produto : compoe

    clientes {
        int idCliente PK
        int qtdePontos
        datetime DtCriacao
        datetime DtAtualizacao
        int flTwitch
        int flYouTube
        int flEmail
        int flBlueSky
        int flInstagram
    }

    transacoes {
        int IdTransacao PK
        int IdCliente FK
        int QtdePontos
        datetime DtCriacao
        string DescSistemaOrigem
    }

    transacao_produto {
        int idTransacaoProduto PK
        int IdTransacao FK
        int IdProduto FK
        int QtdeProduto
        float vlProduto
    }

    produtos {
        int IdProduto PK
        string DescNomeProduto
        string DescDescricaoProduto
        string DescCategoriaProduto
    }
````

---

