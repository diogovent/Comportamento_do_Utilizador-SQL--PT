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

transacoes                clientes          transacao_produto    produtos
     │                       │                      │               │
     ▼                       ▼                      └──────┬────────┘
tb_transações           tb_cliente                         ▼
     │                       │               tb_transação_produto
     ├───────────────────────┤                      │
     ▼                       │               tb_cliente_produto
tb_sumário_transações        │                      │
     │                       │               tb_cliente_produto_rn
     │                       │                      │
     ▼                       │               ┌──────┘
tb_cliente_dia               │               │
     │                       │               │
tb_cliente_dia_rn            │               │
     │                       │               │
tb_cliente_periodo           │               │
     │                       │               │
tb_cliente_periodo_rn        │               │
     │                       │               │
     └───────────────────────┴───────────────┘
                             │
                         tb_join
                             │
                      SELECT final
                    (+ Engajamento_28_Vida)
