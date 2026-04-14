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
