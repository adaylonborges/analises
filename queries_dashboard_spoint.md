# Queries de Construção dos Indicadores do Dashboard Spoint

Abaixo estão as consultas SQL (padrão BigQuery) estruturadas para calcular as métricas apresentadas no Dashboard Analítico do Spoint. Elas estão organizadas conforme as sessões do painel.

*Nota: Os nomes das tabelas (`projeto.dataset...`) são ilustrativos e devem ser ajustados para o mapeamento real do seu Data Lake/Data Warehouse.*

---

## 1. Cadastros e Usuários Ativos (MAU)

### 1.1 Total de Usuários Cadastrados por Origem
Essa query extrai o número total de usuários cadastrados no Spoint, agrupados por qual aplicativo foi a origem do cadastro.

```sql
SELECT 
    nom_origem_cadastro,
    COUNT(DISTINCT cod_spoint_id) as total_usuarios
FROM `ventures-data-lake.refined.vnt_ref_gld_flt_spoint_resumo_usuarios`
WHERE dat_chave = (SELECT MAX(dat_chave) FROM `ventures-data-lake.refined.vnt_ref_gld_flt_spoint_resumo_usuarios`)
GROUP BY nom_origem_cadastro
ORDER BY total_usuarios DESC;
```

### 1.2 Monthly Active Users (MAU) Spoint vs App Centauro
Calcula a média mensal de usuários ativos, permitindo encontrar a penetração do Spoint em relação ao uso do app principal.

```sql
WITH MauSpoint AS (
    SELECT 
        'Spoint' as plataforma,
        'Crava/Spoint' as origem,
        COUNT(DISTINCT cod_spoint_id) as usuarios_unicos,
        EXTRACT(MONTH FROM dat_referencia) as mes_referencia
    FROM `ventures-data-lake.refined.vnt_ref_gld_flt_spoint_resumo_usuarios`
    WHERE flg_acessou = 'S' AND dat_referencia >= '2022-01-01'
    GROUP BY origem, mes_referencia
),
MauCentauro AS (
    SELECT 
        'App Centauro' as plataforma,
        'Geral' as origem,
        COUNT(DISTINCT cod_usuario_pseudonimo) as usuarios_unicos,
        EXTRACT(MONTH FROM dat_chave) as mes_referencia
    FROM `cnto-data-lake.refined.cnt_ref_gld_fat_ga4_navegacao`
    WHERE dat_chave >= '2022-01-01'
      AND nom_plataforma IN ('ANDROID', 'IOS')
    GROUP BY mes_referencia
)

SELECT 
    plataforma,
    origem,
    AVG(usuarios_unicos) as media_mau
FROM (
    SELECT * FROM MauSpoint
    UNION ALL
    SELECT * FROM MauCentauro
)
GROUP BY plataforma, origem;
```

---

## 2. Receita Total e Atribuição (Share)

Calcula a receita gerada no digital agrupada por quem usou benefícios do Spoint na compra, quem tem a conta mas não usou benefício, e quem é apenas da base da Centauro.

```sql
WITH ClientesOutliers AS (
    SELECT cod_cliente
    FROM `ventures-data-lake.refined.vnt_ref_tab_fat_crava_cnto_vendas`
    WHERE dat_captacao >= '2022-01-01'
      AND cod_tipo_movimento = 'V'
      AND flg_megaloja = 'N'
      AND flg_cancelamento_item = 'N' 
      AND flg_compra_pos_crava = 'S'
    GROUP BY cod_cliente
    HAVING COUNT(DISTINCT num_pedido) > 52 OR MAX(vlr_venda_paga) > 5000
)
SELECT 
    CASE 
        WHEN nom_origem_pedido != 'usuario' THEN 'Atribuição Spoint'
        WHEN nom_origem_pedido = 'usuario' THEN 'Sem Atribuição Spoint'
    END as grupo_atribuicao,
    COUNT(DISTINCT cod_cliente) as qtd_clientes
FROM `ventures-data-lake.refined.vnt_ref_tab_fat_crava_cnto_vendas`
WHERE dat_captacao >= '2022-01-01'
  AND cod_tipo_movimento = 'V'
  AND flg_megaloja = 'N'
  AND flg_cancelamento_item = 'N' 
  AND flg_compra_pos_crava = 'S'
  AND cod_cliente NOT IN (SELECT cod_cliente FROM ClientesOutliers)
GROUP BY grupo_atribuicao;
```

```sql
WITH ClientesOutliersSpoint AS (
    SELECT cod_cliente
    FROM `ventures-data-lake.refined.vnt_ref_tab_fat_crava_cnto_vendas`
    WHERE dat_captacao >= '2022-01-01'
      AND cod_tipo_movimento = 'V'
      AND flg_megaloja = 'N'
      AND flg_cancelamento_item = 'N' 
      AND flg_compra_pos_crava = 'S'
    GROUP BY cod_cliente
    HAVING COUNT(DISTINCT num_pedido) > 52 OR MAX(vlr_venda_paga) > 5000
),
ClientesOutliersCentauro AS (
    SELECT cod_cliente
    FROM `cnto-data-lake.refined.cnt_ref_gld_flt_venda_consolidada`
    WHERE dat_captacao >= '2022-01-01'
      AND des_origem_venda = 'DIGITAL'
      AND cod_tipo_movimento = 'V'
    GROUP BY cod_cliente
    HAVING COUNT(DISTINCT num_pedido) > 52 OR MAX(vlr_venda_paga) > 5000
),
ReceitaSpoint AS (
    SELECT 
        EXTRACT(YEAR FROM dat_captacao) as ano,
        CASE 
            WHEN nom_origem_pedido != 'usuario' THEN 'Atribuição Spoint'
            WHEN nom_origem_pedido = 'usuario' THEN 'Sem Atribuição Spoint'
        END as grupo_atribuicao,
        SUM(vlr_venda_paga) as receita_total
    FROM `ventures-data-lake.refined.vnt_ref_tab_fat_crava_cnto_vendas`
    WHERE dat_captacao >= '2022-01-01'
      AND cod_tipo_movimento = 'V'
      AND flg_megaloja = 'N'
      AND flg_cancelamento_item = 'N' 
      AND flg_compra_pos_crava = 'S'
      AND nom_origem_pedido IS NOT NULL 
      AND cod_cliente NOT IN (SELECT cod_cliente FROM ClientesOutliersSpoint)
    GROUP BY ano, grupo_atribuicao
),
ReceitaCentauro AS (
    SELECT 
        EXTRACT(YEAR FROM dat_captacao) as ano,
        'Centauro (100% Geral)' as grupo_atribuicao,
        SUM(vlr_venda_paga) as receita_total
    FROM `cnto-data-lake.refined.cnt_ref_gld_flt_venda_consolidada`
    WHERE dat_captacao >= '2022-01-01'
      AND des_origem_venda = 'DIGITAL'
      AND cod_tipo_movimento = 'V'
      AND cod_cliente NOT IN (SELECT cod_cliente FROM ClientesOutliersCentauro)
    GROUP BY ano
)
SELECT * FROM ReceitaSpoint
UNION ALL
SELECT * FROM ReceitaCentauro
ORDER BY ano, receita_total DESC;
```

---

## 3. Comportamento de Base e Aquisição

Descobre quantos usuários conheceram a marca por conta do Spoint (cadastraram no Spoint antes da primeira compra na Centauro) versus os que já eram clientes históricos.

```sql
WITH ClientesOutliersCentauro AS (
    SELECT cod_cliente
    FROM `cnto-data-lake.refined.cnt_ref_gld_flt_venda_consolidada`
    WHERE dat_captacao >= '2022-01-01'
      AND des_origem_venda = 'DIGITAL'
      AND cod_tipo_movimento = 'V'
    GROUP BY cod_cliente
    HAVING COUNT(DISTINCT num_pedido) > 52 OR MAX(vlr_venda_paga) > 5000
),
PrimeiraCompra AS (
    SELECT cod_cliente, MIN(dat_captacao) as data_primeira_compra
    FROM `cnto-data-lake.refined.cnt_ref_gld_flt_venda_consolidada`
    WHERE des_origem_venda = 'DIGITAL' 
      AND cod_tipo_movimento = 'V'
      AND cod_cliente NOT IN (SELECT cod_cliente FROM ClientesOutliersCentauro)
    GROUP BY cod_cliente
),
CadastroSpoint AS (
    SELECT cod_usuario_cdp as user_id, dat_criacao_usuario as data_cadastro
    FROM `ventures-data-lake.refined.vnt_ref_gld_dim_crava_usuarios`
)
SELECT 
    CASE 
        WHEN DATE(s.data_cadastro) <= DATE(c.data_primeira_compra) THEN 'Novos clientes (influência do Spoint)'
        WHEN c.data_primeira_compra IS NULL THEN 'Novos cadastros (Ainda não compraram)'
        ELSE 'Base Histórica (Já eram clientes)'
    END as tipo_cliente,
    COUNT(DISTINCT s.user_id) as total_usuarios_spoint,
    COUNT(DISTINCT CASE WHEN c.data_primeira_compra IS NOT NULL THEN s.user_id END) as total_compradores_ativos
FROM CadastroSpoint s
LEFT JOIN PrimeiraCompra c ON CAST(s.user_id AS STRING) = c.cod_cliente
GROUP BY tipo_cliente;
```

---

## 4. Comprovação do Lift (Receita de Influência do Programa)

Calcula a matemática de prova, comparando o comportamento da Base Orgânica com o Grupo do Spoint para achar a receita puramente adicional gerada pela retenção.

```sql
WITH ClientesOutliersSpoint AS (
    SELECT cod_cliente
    FROM `ventures-data-lake.refined.vnt_ref_tab_fat_crava_cnto_vendas`
    WHERE dat_captacao >= '2022-01-01'
      AND cod_tipo_movimento = 'V'
      AND flg_megaloja = 'N'
      AND flg_cancelamento_item = 'N' 
      AND flg_compra_pos_crava = 'S'
    GROUP BY cod_cliente
    HAVING COUNT(DISTINCT num_pedido) > 52 OR MAX(vlr_venda_paga) > 5000
),
ClientesOutliersCentauro AS (
    SELECT cod_cliente
    FROM `cnto-data-lake.refined.cnt_ref_gld_flt_venda_consolidada`
    WHERE dat_captacao >= '2022-01-01'
      AND des_origem_venda = 'DIGITAL'
      AND cod_tipo_movimento = 'V'
    GROUP BY cod_cliente
    HAVING COUNT(DISTINCT num_pedido) > 52 OR MAX(vlr_venda_paga) > 5000
),
VendasSpoint AS (
    SELECT 
        CASE 
            WHEN nom_origem_pedido != 'usuario' THEN 'Tratamento (Spoint - Com Atribuição)'
            WHEN nom_origem_pedido = 'usuario' THEN 'Tratamento (Spoint - Sem Atribuição)'
        END as grupo,
        cod_cliente,
        CAST(num_pedido AS STRING) as num_pedido,
        CAST(vlr_venda_paga AS FLOAT64) as vlr_venda_paga
    FROM `ventures-data-lake.refined.vnt_ref_tab_fat_crava_cnto_vendas`
    WHERE dat_captacao >= '2022-01-01'
      AND cod_cliente NOT IN (SELECT cod_cliente FROM ClientesOutliersSpoint)
),
ClientesSpoint AS (
    SELECT DISTINCT cod_cliente FROM VendasSpoint
),
VendasCentauro AS (
    SELECT 
        'Controle (Centauro Geral)' as grupo,
        cod_cliente,
        CAST(num_pedido AS STRING) as num_pedido,
        CAST(vlr_venda_paga AS FLOAT64) as vlr_venda_paga
    FROM `cnto-data-lake.refined.cnt_ref_gld_flt_venda_consolidada`
    WHERE dat_captacao >= '2022-01-01'
      AND des_origem_venda = 'DIGITAL'
      AND cod_tipo_movimento = 'V'
      AND cod_cliente NOT IN (SELECT cod_cliente FROM ClientesSpoint)
      AND cod_cliente NOT IN (SELECT cod_cliente FROM ClientesOutliersCentauro)
),
Consolidado AS (
    SELECT * FROM VendasSpoint
    UNION ALL
    SELECT * FROM VendasCentauro
),
Agrupado AS (
    SELECT 
        grupo,
        COUNT(DISTINCT cod_cliente) as clientes_unicos,
        COUNT(DISTINCT num_pedido) / COUNT(DISTINCT cod_cliente) as frequencia_media,
        SUM(vlr_venda_paga) / COUNT(DISTINCT num_pedido) as ticket_medio,
        SUM(vlr_venda_paga) as receita_real
    FROM Consolidado
    WHERE grupo IS NOT NULL
    GROUP BY grupo
),
ControleStats AS (
    SELECT frequencia_media, ticket_medio 
    FROM Agrupado 
    WHERE grupo = 'Controle (Centauro Geral)'
)
SELECT 
    t.grupo,
    t.clientes_unicos,
    t.frequencia_media as frequencia_real,
    t.ticket_medio as ticket_real,
    t.receita_real,
    (t.clientes_unicos * c.frequencia_media * c.ticket_medio) as projecao_organica_sem_programa,
    (t.receita_real - (t.clientes_unicos * c.frequencia_media * c.ticket_medio)) as receita_incremental_lift
FROM Agrupado t
CROSS JOIN ControleStats c
WHERE t.grupo != 'Controle (Centauro Geral)';
```

---

## 5. Matriz de Comportamento por Tier (Tiers de Receita/LTV)

A principal e mais complexa query do Dashboard. Rankeia os usuários da base digital em decis de valor (Top 10%, Top 25%, etc) utilizando a regra oficial de Lucro Bruto (Margem) da Centauro para todo o período selecionado. Esta query também cruza com a retenção (recompra) em M3 e M6, retirando outliers de altíssima frequência.

```sql
WITH clientes_puro_digital AS (
  SELECT
    cod_cliente_cdp
  FROM `cnto-data-lake.refined.cnt_ref_gld_fat_cliente_comportamento_compra`
  WHERE
    dat_chave BETWEEN '2025-01-01' AND '2026-12-31'
    AND flg_megaloja = 'N'
    AND cod_cliente_cdp IS NOT NULL
    AND cod_cliente_cdp != '-1'
  GROUP BY 1
  HAVING
    COUNTIF(LOWER(des_origem_venda) = 'canal_ecom') > 0
    AND COUNTIF(LOWER(des_origem_venda) != 'canal_ecom') = 0
),

clientes_spoint AS (
  SELECT 
    dim.cod_cliente_cdp,
    MAX(CASE WHEN v.nom_origem_pedido != 'usuario' THEN 1 ELSE 0 END) as flag_com_atribuicao,
    MAX(CASE WHEN v.nom_origem_pedido = 'usuario' THEN 1 ELSE 0 END) as flag_sem_atribuicao
  FROM `ventures-data-lake.refined.vnt_ref_tab_fat_crava_cnto_vendas` v
  INNER JOIN `cnto-data-lake.refined.cnt_ref_gld_dim_cliente` dim
    ON CAST(v.cod_usuario_site AS STRING) = dim.cod_cnt_ecom
  WHERE EXTRACT(YEAR FROM v.dat_pagamento) IN (2025, 2026)
  GROUP BY 1
),

compras_com_proxima AS (
  SELECT 
    c.cod_cliente_cdp,
    EXTRACT(YEAR FROM c.dat_chave) AS ano,
    c.dat_chave,
    LEAD(c.dat_chave) OVER (PARTITION BY c.cod_cliente_cdp ORDER BY c.dat_chave) as proxima_compra
  FROM `cnto-data-lake.refined.cnt_ref_gld_fat_cliente_comportamento_compra` c
  INNER JOIN clientes_puro_digital b ON c.cod_cliente_cdp = b.cod_cliente_cdp
  WHERE c.dat_chave >= '2022-01-01'
    AND c.flg_megaloja = 'N'
    AND c.cod_cliente_cdp IS NOT NULL
    AND c.cod_cliente_cdp != '-1'
    AND LOWER(c.des_origem_venda) = 'canal_ecom'
),

recompra_total AS (
  SELECT
    cod_cliente_cdp,
    MAX(CASE WHEN DATE_DIFF(proxima_compra, dat_chave, DAY) > 0 AND DATE_DIFF(proxima_compra, dat_chave, DAY) <= 90 THEN 1 ELSE 0 END) as flg_rec_m3,
    MAX(CASE WHEN DATE_DIFF(proxima_compra, dat_chave, DAY) > 0 AND DATE_DIFF(proxima_compra, dat_chave, DAY) <= 180 THEN 1 ELSE 0 END) as flg_rec_m6
  FROM compras_com_proxima
  WHERE ano IN (2025, 2026)
  GROUP BY 1
),

comportamento_anual AS (
  SELECT
    c.cod_cliente_cdp,
    EXTRACT(YEAR FROM c.dat_chave) AS ano,
    COUNT(DISTINCT c.num_pedido) AS pedidos_ano,
    SUM(c.qtd_itens_total)       AS itens_ano,
    SUM(c.vlr_venda_paga_total)  AS spending_ano,
    MAX(c.vlr_venda_paga_total)  AS max_ticket_pedido
  FROM `cnto-data-lake.refined.cnt_ref_gld_fat_cliente_comportamento_compra` c
  INNER JOIN clientes_puro_digital b
    ON c.cod_cliente_cdp = b.cod_cliente_cdp
  WHERE
    c.dat_chave BETWEEN '2025-01-01' AND '2026-12-31'
    AND c.flg_megaloja = 'N'
    AND c.cod_cliente_cdp IS NOT NULL
    AND c.cod_cliente_cdp != '-1'
    AND LOWER(c.des_origem_venda) = 'canal_ecom'
  GROUP BY 1, 2
),

comportamento AS (
  SELECT
    cod_cliente_cdp,
    SUM(pedidos_ano)                              AS pedidos_total,
    AVG(pedidos_ano)                              AS frequencia_media_ano,
    AVG(spending_ano)                             AS spending_medio_ano,
    SAFE_DIVIDE(SUM(itens_ano), SUM(pedidos_ano)) AS itens_por_pedido
  FROM comportamento_anual
  GROUP BY 1
  -- REGRA DE OUTLIERS
  HAVING SUM(pedidos_ano) <= 52 
     AND MAX(max_ticket_pedido) <= 5000
),

margem AS (
  SELECT
    v.cod_cliente_cdp_vigente,
    SUM(v.vlr_venda_liquida) AS rol,
    SUM(
      CASE
        WHEN LOWER(v.des_marca) != 'nike' THEN v.vlr_custo_empresa + v.vlr_custo_fiscal
        ELSE v.vlr_custo_empresa_fisia
          + v.vlr_custo_fiscal_fisia
          + v.vlr_royalties_custo_fisia
      END
    ) AS cmv
  FROM `cnto-data-lake.refined.cnt_ref_gld_flt_venda_consolidada` v
  INNER JOIN clientes_puro_digital b
    ON v.cod_cliente_cdp_vigente = b.cod_cliente_cdp
  WHERE
    v.dat_chave BETWEEN '2025-01-01' AND '2026-12-31'
    AND v.flg_cancelamento_item = 'N'
    AND v.flg_cancelamento_pedido = 'N'
    AND v.flg_faturamento = 'S'
    AND v.flg_megaloja = 'N'
    AND v.cod_tipo_movimento = 'V'
    AND v.dat_faturamento IS NOT NULL
    AND REGEXP_CONTAINS(v.cod_produto, r'^[0-9]')
    AND v.cod_cliente_cdp_vigente IS NOT NULL
    AND v.cod_cliente_cdp_vigente != '-1'
    AND UPPER(v.des_origem_venda) = 'DIGITAL'
  GROUP BY 1
),

base AS (
  SELECT
    c.cod_cliente_cdp,
    CASE 
      WHEN s.flag_com_atribuicao = 1 THEN 'Atribuição Spoint'
      WHEN s.flag_sem_atribuicao = 1 THEN 'Sem atribuição Spoint'
      ELSE 'Outros Centauro'
    END AS grupo_spoint,
    m.rol                                                     AS ROL,
    m.cmv                                                     AS CMV,
    (m.rol - m.cmv)                                           AS lucro_bruto,
    SAFE_DIVIDE(m.rol - m.cmv, m.rol)                         AS margem_bruta,
    c.pedidos_total,
    c.frequencia_media_ano,
    c.spending_medio_ano,
    c.itens_por_pedido,
    SAFE_DIVIDE(c.spending_medio_ano, c.frequencia_media_ano) AS ticket_medio,
    COALESCE(r.flg_rec_m3, 0)                                 AS flg_rec_m3,
    COALESCE(r.flg_rec_m6, 0)                                 AS flg_rec_m6
  FROM comportamento c
  LEFT JOIN margem m
    ON c.cod_cliente_cdp = m.cod_cliente_cdp_vigente
  LEFT JOIN clientes_spoint s 
    ON c.cod_cliente_cdp = s.cod_cliente_cdp
  LEFT JOIN recompra_total r
    ON c.cod_cliente_cdp = r.cod_cliente_cdp
  WHERE m.rol IS NOT NULL
),

base_com_ranking AS (
  SELECT
    *,
    RANK() OVER (ORDER BY lucro_bruto DESC) AS ranking_lucro,
    COUNT(*) OVER ()                        AS total_clientes
  FROM base
),

base_tiers AS (
  SELECT
    *,
    CASE
      WHEN SAFE_DIVIDE(ranking_lucro, total_clientes) <= 0.10 THEN 'Top 10%'
      WHEN SAFE_DIVIDE(ranking_lucro, total_clientes) <= 0.25 THEN 'Top 25%'
      WHEN SAFE_DIVIDE(ranking_lucro, total_clientes) <= 0.50 THEN 'Top 50%'
      WHEN SAFE_DIVIDE(ranking_lucro, total_clientes) <= 0.75 THEN 'Top 75%'
      WHEN SAFE_DIVIDE(ranking_lucro, total_clientes) <= 0.90 THEN 'Top 90%'
      ELSE 'Bottom 10%'
    END AS tier
  FROM base_com_ranking
)

-- Agregação final para o Dashboard
SELECT
  tier,
  'Centauro' AS grupo,
  COUNT(DISTINCT cod_cliente_cdp) AS qtd_clientes,
  AVG(lucro_bruto) AS media_lucro_bruto,
  AVG(frequencia_media_ano) AS frequencia_media,
  AVG(spending_medio_ano) AS spending_medio,
  AVG(ticket_medio) AS ticket_medio,
  AVG(itens_por_pedido) AS itens_por_pedido,
  SAFE_DIVIDE(SUM(lucro_bruto), SUM(ROL)) * 100 AS margem_bruta_pct,
  SAFE_DIVIDE(SUM(flg_rec_m3), COUNT(DISTINCT cod_cliente_cdp)) * 100 AS taxa_rec_m3,
  SAFE_DIVIDE(SUM(flg_rec_m6), COUNT(DISTINCT cod_cliente_cdp)) * 100 AS taxa_rec_m6
FROM base_tiers
GROUP BY 1

UNION ALL

SELECT
  tier,
  grupo_spoint AS grupo,
  COUNT(DISTINCT cod_cliente_cdp) AS qtd_clientes,
  AVG(lucro_bruto) AS media_lucro_bruto,
  AVG(frequencia_media_ano) AS frequencia_media,
  AVG(spending_medio_ano) AS spending_medio,
  AVG(ticket_medio) AS ticket_medio,
  AVG(itens_por_pedido) AS itens_por_pedido,
  SAFE_DIVIDE(SUM(lucro_bruto), SUM(ROL)) * 100 AS margem_bruta_pct,
  SAFE_DIVIDE(SUM(flg_rec_m3), COUNT(DISTINCT cod_cliente_cdp)) * 100 AS taxa_rec_m3,
  SAFE_DIVIDE(SUM(flg_rec_m6), COUNT(DISTINCT cod_cliente_cdp)) * 100 AS taxa_rec_m6
FROM base_tiers
WHERE grupo_spoint IN ('Atribuição Spoint', 'Sem atribuição Spoint')
GROUP BY 1, 2

ORDER BY tier, grupo;
```

---

## 6. Estudo Comportamental AS/DS (Antes e Depois do Spoint)

Nesta visão avançada de CRM, abandonamos o calendário linear. O "Marco Zero" passa a ser a **Data de Adesão ao Spoint** de cada cliente (a sua "Safra Spoint").
A partir daí, olhamos para a vida do cliente em *Quarters Relativos*:
- **Q-3 a Q-1:** O comportamento orgânico dele na Centauro nos meses anteriores à adesão.
- **Q+1 a Q+3:** O comportamento estimulado dele na Centauro após a adesão.

A query também classifica a origem do cliente, separando quem já era comprador ativo da Centauro de quem fez a sua primeira compra apenas por causa do Spoint.

```sql
WITH ClientesOutliersCentauro AS (
    SELECT cod_cliente
    FROM `cnto-data-lake.refined.cnt_ref_gld_flt_venda_consolidada`
    WHERE dat_captacao >= '2022-01-01'
      AND des_origem_venda = 'DIGITAL'
      AND cod_tipo_movimento = 'V'
    GROUP BY cod_cliente
    HAVING COUNT(DISTINCT num_pedido) > 52 OR MAX(vlr_venda_paga) > 5000
),
ClientesSpoint AS (
    SELECT 
        cod_usuario_cdp as cod_cliente,
        CAST(dat_criacao_usuario AS DATE) as data_entrada_spoint,
        CONCAT(EXTRACT(YEAR FROM dat_criacao_usuario), '-Q', EXTRACT(QUARTER FROM dat_criacao_usuario)) as safra_spoint
    FROM `ventures-data-lake.refined.vnt_ref_gld_dim_crava_usuarios`
    WHERE cod_usuario_cdp IS NOT NULL AND cod_usuario_cdp != '-1'
      AND dat_criacao_usuario >= '2022-01-01'
      AND cod_usuario_cdp NOT IN (SELECT CAST(cod_cliente AS STRING) FROM ClientesOutliersCentauro)
),
PrimeiraCompra AS (
    SELECT 
        cod_cliente, 
        MIN(CAST(dat_captacao AS DATE)) as data_primeira_compra
    FROM `cnto-data-lake.refined.cnt_ref_gld_flt_venda_consolidada`
    WHERE des_origem_venda = 'DIGITAL' 
      AND cod_tipo_movimento = 'V'
      AND cod_cliente NOT IN (SELECT cod_cliente FROM ClientesOutliersCentauro)
    GROUP BY cod_cliente
),
PerfilCliente AS (
    SELECT 
        s.cod_cliente,
        s.data_entrada_spoint,
        s.safra_spoint,
        p.data_primeira_compra,
        CASE 
            WHEN p.data_primeira_compra < s.data_entrada_spoint THEN 'Antigo Centauro' 
            ELSE 'Novo via Spoint' 
        END as flg_perfil
    FROM ClientesSpoint s
    LEFT JOIN PrimeiraCompra p ON s.cod_cliente = CAST(p.cod_cliente AS STRING)
),

TodasVendas AS (
    SELECT 
        v.cod_cliente,
        v.num_pedido,
        v.vlr_venda_paga,
        p.safra_spoint,
        DATE_DIFF(CAST(v.dat_captacao AS DATE), p.data_entrada_spoint, DAY) as diff_dias
    FROM `cnto-data-lake.refined.cnt_ref_gld_flt_venda_consolidada` v
    INNER JOIN PerfilCliente p ON v.cod_cliente = p.cod_cliente
    WHERE v.des_origem_venda = 'DIGITAL'
      AND v.cod_tipo_movimento = 'V'
      AND v.flg_faturamento = 'S'
      AND v.flg_cancelamento_pedido = 'N'
      AND v.flg_cancelamento_item = 'N'
      AND p.flg_perfil = 'Antigo Centauro'
      AND v.cod_cliente NOT IN (SELECT cod_cliente FROM ClientesOutliersCentauro)
),
FiltroElegibilidade AS (
    SELECT cod_cliente, safra_spoint
    FROM TodasVendas
    GROUP BY cod_cliente, safra_spoint
    HAVING COUNT(DISTINCT CASE WHEN diff_dias BETWEEN -365 AND -1 THEN num_pedido END) >= 1
       AND COUNT(DISTINCT CASE WHEN diff_dias BETWEEN 0 AND 365 THEN num_pedido END) >= 1
),
BaseSafrasElegiveis AS (
    SELECT safra_spoint, COUNT(DISTINCT cod_cliente) as total_antigos_elegiveis
    FROM FiltroElegibilidade
    GROUP BY safra_spoint
),
AgregadoFrequencia AS (
    SELECT 
        safra_spoint,
        COUNT(DISTINCT CASE WHEN diff_dias BETWEEN -90 AND -1 THEN num_pedido END) as pedidos_3m_antes,
        COUNT(DISTINCT CASE WHEN diff_dias BETWEEN 0 AND 90 THEN num_pedido END) as pedidos_3m_depois,
        COUNT(DISTINCT CASE WHEN diff_dias BETWEEN -180 AND -1 THEN num_pedido END) as pedidos_6m_antes,
        COUNT(DISTINCT CASE WHEN diff_dias BETWEEN 0 AND 180 THEN num_pedido END) as pedidos_6m_depois,
        COUNT(DISTINCT CASE WHEN diff_dias BETWEEN -365 AND -1 THEN num_pedido END) as pedidos_12m_antes,
        COUNT(DISTINCT CASE WHEN diff_dias BETWEEN 0 AND 365 THEN num_pedido END) as pedidos_12m_depois,
        SUM(CASE WHEN diff_dias BETWEEN -90 AND -1 THEN vlr_venda_paga ELSE 0 END) as spending_3m_antes,
        SUM(CASE WHEN diff_dias BETWEEN 0 AND 90 THEN vlr_venda_paga ELSE 0 END) as spending_3m_depois,
        SUM(CASE WHEN diff_dias BETWEEN -180 AND -1 THEN vlr_venda_paga ELSE 0 END) as spending_6m_antes,
        SUM(CASE WHEN diff_dias BETWEEN 0 AND 180 THEN vlr_venda_paga ELSE 0 END) as spending_6m_depois,
        SUM(CASE WHEN diff_dias BETWEEN -365 AND -1 THEN vlr_venda_paga ELSE 0 END) as spending_12m_antes,
        SUM(CASE WHEN diff_dias BETWEEN 0 AND 365 THEN vlr_venda_paga ELSE 0 END) as spending_12m_depois
    FROM TodasVendas v
    INNER JOIN FiltroElegibilidade f ON v.cod_cliente = f.cod_cliente
    GROUP BY v.safra_spoint
)
SELECT 
    b.safra_spoint,
    b.total_antigos_elegiveis as total_antigos,
    (a.pedidos_3m_antes / b.total_antigos_elegiveis) as media_pedidos_3m_antes,
    (a.pedidos_3m_depois / b.total_antigos_elegiveis) as media_pedidos_3m_depois,
    (a.pedidos_6m_antes / b.total_antigos_elegiveis) as media_pedidos_6m_antes,
    (a.pedidos_6m_depois / b.total_antigos_elegiveis) as media_pedidos_6m_depois,
    (a.pedidos_12m_antes / b.total_antigos_elegiveis) as media_pedidos_12m_antes,
    (a.pedidos_12m_depois / b.total_antigos_elegiveis) as media_pedidos_12m_depois,
    (a.spending_3m_antes / b.total_antigos_elegiveis) as media_spending_3m_antes,
    (a.spending_3m_depois / b.total_antigos_elegiveis) as media_spending_3m_depois,
    (a.spending_6m_antes / b.total_antigos_elegiveis) as media_spending_6m_antes,
    (a.spending_6m_depois / b.total_antigos_elegiveis) as media_spending_6m_depois,
    (a.spending_12m_antes / b.total_antigos_elegiveis) as media_spending_12m_antes,
    (a.spending_12m_depois / b.total_antigos_elegiveis) as media_spending_12m_depois
FROM BaseSafrasElegiveis b
LEFT JOIN AgregadoFrequencia a ON b.safra_spoint = a.safra_spoint
ORDER BY b.safra_spoint;
```
