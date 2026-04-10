# Dicionário de Termos de Análise de Dados
## Referência Completa para Analistas

---

## Índice por Categoria

1. [Engenharia de Dados](#engenharia-de-dados)
2. [Modelagem de Dados](#modelagem-de-dados)
3. [Estatística e Probabilidade](#estatistica-e-probabilidade)
4. [Análise Exploratória de Dados](#analise-exploratoria)
5. [Métricas de Negócio](#metricas-de-negocio)
6. [Segmentação e Machine Learning](#segmentacao-ml)
7. [Visualização e BI](#visualizacao-bi)
8. [Ferramentas e Tecnologias](#ferramentas)
9. [Siglas e Abreviações](#siglas)

---

## 1. Engenharia de Dados {#engenharia-de-dados}

### **ETL (Extract, Transform, Load)**
**Extração, Transformação e Carga**

**Definição**: Processo de engenharia de dados que envolve extrair dados de fontes diversas, transformá-los em um formato consistente e carregá-los em um destino (geralmente um Data Warehouse).

**Componentes**:
1. **Extract**: Captura de dados de sistemas fonte (bancos de dados, APIs, arquivos)
2. **Transform**: Limpeza, validação, agregação e enriquecimento
3. **Load**: Inserção no destino final

**Exemplo Prático**:
```sql
-- Extract
SELECT * FROM Producao.dbo.Pedidos 
WHERE data_atualizacao > @ultima_carga;

-- Transform (limpeza de outliers)
WHERE valor_total BETWEEN 0 AND 100000;

-- Load
INSERT INTO DW.dbo.Fact_Pedidos (...) 
VALUES (...);
```

**Quando usar**: Quando você precisa consolidar dados de múltiplas fontes em um repositório centralizado para análise. Ideal para cargas batch (noturnas, horárias).

**Relacionado**: ELT, Data Pipeline, Data Integration

---

### **ELT (Extract, Load, Transform)**
**Extração, Carga e Transformação**

**Definição**: Variação do ETL onde os dados são carregados no destino ANTES da transformação, aproveitando o poder de processamento do Data Warehouse moderno.

**Diferença do ETL**:
- **ETL**: Transformação em servidor intermediário → Load
- **ELT**: Load direto → Transformação no DW (SQL, dbt, etc.)

**Exemplo**:
```sql
-- Load raw data
INSERT INTO Staging.Pedidos_Raw 
SELECT * FROM Source;

-- Transform in-database
CREATE VIEW Pedidos_Clean AS
SELECT 
    pedido_id,
    UPPER(TRIM(status)) AS status,
    CASE WHEN valor_total < 0 THEN 0 ELSE valor_total END AS valor_total
FROM Staging.Pedidos_Raw;
```

**Quando usar**: Em ambientes cloud (BigQuery, Snowflake, Synapse) onde computação é barata e escalável. Permite iteração rápida em transformações.

**Relacionado**: ETL, Modern Data Stack, Cloud Computing

---

### **Data Warehouse (DW)**
**Armazém de Dados**

**Definição**: Repositório centralizado de dados estruturados, otimizado para análise e reporting. Armazena dados históricos de múltiplas fontes em formato dimensional.

**Características**:
- **Subject-Oriented**: Organizado por assuntos de negócio (vendas, clientes, produtos)
- **Integrated**: Dados de múltiplas fontes consolidados
- **Time-Variant**: Mantém histórico
- **Non-Volatile**: Dados não são alterados, apenas adicionados

**Arquitetura Típica**:
```
OLTP (Produção) → ETL → Staging → Data Warehouse → Data Marts → BI Tools
```

**Exemplo de Estrutura**:
```sql
-- Tabela Fato (transações)
CREATE TABLE Fact_Vendas (
    venda_id INT,
    data_key INT,
    cliente_key INT,
    produto_key INT,
    quantidade INT,
    valor_total DECIMAL(10,2)
);

-- Tabela Dimensão (atributos)
CREATE TABLE Dim_Cliente (
    cliente_key INT PRIMARY KEY,
    cliente_id INT,
    nome VARCHAR(200),
    segmento VARCHAR(50),
    data_inicio DATE,
    data_fim DATE,
    is_current BIT
);
```

**Quando usar**: Para análises históricas, relatórios complexos e consolidação de dados de múltiplos sistemas. Essencial para empresas que precisam de single source of truth.

**Relacionado**: OLAP, Data Lake, Star Schema, Kimball Methodology

---

### **Data Lake**
**Lago de Dados**

**Definição**: Repositório centralizado que armazena dados estruturados e não-estruturados em seu formato nativo (raw). Diferente do Data Warehouse, não exige schema pré-definido.

**Dados armazenados**:
- Estruturados: CSVs, tabelas SQL
- Semi-estruturados: JSON, XML, logs
- Não-estruturados: imagens, vídeos, PDFs, texto livre

**Arquitetura de Camadas**:
```
Bronze (Raw) → Silver (Cleaned) → Gold (Business-Level Aggregated)
```

**Exemplo**:
```
s3://data-lake/
├── bronze/
│   ├── vendas_raw/2024/01/01/vendas.parquet
│   └── logs_app/2024/01/01/logs.json
├── silver/
│   ├── vendas_cleaned/2024/01/vendas.parquet
└── gold/
    └── vendas_agregadas_diarias/2024/01/summary.parquet
```

**Quando usar**: Quando você tem grande volume de dados diversos (estruturados + não-estruturados), precisa de flexibilidade de schema, ou quer fazer machine learning com dados brutos.

**Relacionado**: Data Warehouse, Big Data, Hadoop, Spark, Parquet

---

### **OLTP (Online Transaction Processing)**
**Processamento de Transações Online**

**Definição**: Sistema otimizado para operações transacionais do dia a dia (inserir, atualizar, deletar). Prioriza velocidade e consistência de escritas.

**Características**:
- **Normalizado**: Minimiza redundância (3NF ou superior)
- **ACID Compliant**: Atomicidade, Consistência, Isolamento, Durabilidade
- **Operações**: INSERT, UPDATE, DELETE frequentes
- **Queries**: Simples, focadas em poucos registros

**Exemplo de Schema OLTP**:
```sql
-- Altamente normalizado
CREATE TABLE Clientes (
    cliente_id INT PRIMARY KEY,
    nome VARCHAR(200)
);

CREATE TABLE Enderecos (
    endereco_id INT PRIMARY KEY,
    cliente_id INT FOREIGN KEY REFERENCES Clientes,
    rua VARCHAR(200),
    cidade VARCHAR(100)
);

CREATE TABLE Pedidos (
    pedido_id INT PRIMARY KEY,
    cliente_id INT FOREIGN KEY REFERENCES Clientes,
    data_pedido DATETIME
);
```

**Quando usar**: Para sistemas operacionais (e-commerce em produção, CRM, ERP). NÃO use para análises complexas.

**Relacionado**: OLAP, Normalização, ACID

---

### **OLAP (Online Analytical Processing)**
**Processamento Analítico Online**

**Definição**: Sistema otimizado para consultas analíticas complexas e agregações. Prioriza velocidade de leitura e facilidade de análise.

**Características**:
- **Desnormalizado**: Dados redundantes para performance (Star/Snowflake Schema)
- **Read-Heavy**: Otimizado para SELECT com agregações
- **Histórico**: Mantém dados ao longo do tempo
- **Queries**: Complexas, com múltiplos JOINs e GROUP BY

**Exemplo de Schema OLAP**:
```sql
-- Star Schema (desnormalizado)
CREATE TABLE Fact_Vendas (
    venda_id INT,
    data_key INT,           -- FK para Dim_Data
    cliente_key INT,        -- FK para Dim_Cliente
    produto_key INT,        -- FK para Dim_Produto
    loja_key INT,           -- FK para Dim_Loja
    quantidade INT,
    valor_total DECIMAL(10,2),
    custo DECIMAL(10,2)
);

CREATE TABLE Dim_Produto (
    produto_key INT PRIMARY KEY,
    produto_id INT,
    nome_produto VARCHAR(200),
    categoria VARCHAR(100),
    subcategoria VARCHAR(100),
    marca VARCHAR(100)
    -- Tudo em uma tabela para facilitar análise
);
```

**Operações OLAP**:
- **Slice**: Filtrar uma dimensão
- **Dice**: Filtrar múltiplas dimensões
- **Drill-Down**: Aumentar granularidade (ano → mês → dia)
- **Roll-Up**: Diminuir granularidade (dia → mês → ano)
- **Pivot**: Rotacionar dimensões

**Quando usar**: Para análises históricas, dashboards, relatórios executivos. Base de um Data Warehouse.

**Relacionado**: OLTP, Data Warehouse, Cubo OLAP, MDX

---

### **CDC (Change Data Capture)**
**Captura de Dados de Mudança**

**Definição**: Técnica para identificar e capturar apenas os dados que foram modificados em um sistema fonte, evitando carga completa (full load).

**Métodos**:
1. **Timestamp-based**: Usa coluna de data/hora de atualização
2. **Log-based**: Lê transaction logs do banco
3. **Trigger-based**: Triggers capturam mudanças em tabela separada

**Exemplo Timestamp-based**:
```sql
-- Tabela fonte com controle de mudança
CREATE TABLE Clientes (
    cliente_id INT PRIMARY KEY,
    nome VARCHAR(200),
    email VARCHAR(100),
    data_atualizacao DATETIME DEFAULT GETDATE()
);

-- Trigger para atualizar timestamp
CREATE TRIGGER trg_Clientes_Update
ON Clientes
AFTER UPDATE
AS
BEGIN
    UPDATE Clientes
    SET data_atualizacao = GETDATE()
    WHERE cliente_id IN (SELECT cliente_id FROM inserted);
END;

-- ETL incremental
DECLARE @ultima_carga DATETIME = '2024-01-01 00:00:00';

SELECT * 
FROM Clientes
WHERE data_atualizacao > @ultima_carga;
```

**Exemplo Log-based (SQL Server)**:
```sql
-- Habilitar CDC
EXEC sys.sp_cdc_enable_db;

EXEC sys.sp_cdc_enable_table
    @source_schema = 'dbo',
    @source_name = 'Clientes',
    @role_name = NULL;

-- Ler mudanças
SELECT *
FROM cdc.dbo_Clientes_CT
WHERE __$start_lsn > @last_lsn
  AND __$operation IN (2, 4); -- 2=INSERT, 4=UPDATE
```

**Quando usar**: Para sincronização em tempo real ou near-real-time entre sistemas. Reduz drasticamente volume de dados transferidos e processados.

**Relacionado**: ETL Incremental, Replicação, Streaming

---

### **Data Pipeline**
**Pipeline de Dados**

**Definição**: Conjunto automatizado de processos que movem dados de uma ou mais fontes para um destino, aplicando transformações no caminho.

**Componentes típicos**:
```
Source → Extract → Validate → Transform → Enrich → Load → Destination
```

**Exemplo com orquestração**:
```sql
-- Step 1: Extract
EXEC sp_Extract_Pedidos;

-- Step 2: Validate
EXEC sp_Validate_Data_Quality;

-- Step 3: Transform
EXEC sp_Transform_Pedidos;

-- Step 4: Load
EXEC sp_Load_To_DW;

-- Step 5: Notify
EXEC sp_Send_Success_Email;
```

**Orquestração**: Ferramentas como SQL Server Agent, Apache Airflow, Azure Data Factory gerenciam dependências e retries.

**Quando usar**: Quando você precisa automatizar fluxos de dados repetitivos. Essencial para garantir consistência e auditoria.

**Relacionado**: ETL/ELT, Workflow Orchestration, DAG (Directed Acyclic Graph)

---

### **Staging Area**
**Área de Preparação**

**Definição**: Camada intermediária no processo ETL onde dados brutos são temporariamente armazenados antes de serem transformados e carregados no Data Warehouse.

**Propósitos**:
1. **Isolamento**: Não impacta sistema fonte (OLTP)
2. **Validação**: Permite verificar qualidade antes de carregar no DW
3. **Recovery**: Backup em caso de falha na transformação
4. **Performance**: Otimiza processamento batch

**Arquitetura**:
```
OLTP → [Staging] → Transform → Data Warehouse
```

**Exemplo**:
```sql
-- Staging schema separado
CREATE SCHEMA Staging;

-- Tabela staging (estrutura simples, sem constraints)
CREATE TABLE Staging.Pedidos (
    pedido_id INT,
    cliente_id INT,
    data_pedido VARCHAR(50),  -- Pode vir em formato inconsistente
    valor_total VARCHAR(50),  -- Pode vir com "$" ou vírgula
    status VARCHAR(50),
    load_timestamp DATETIME DEFAULT GETDATE()
);

-- Processo ETL
-- 1. Truncate staging
TRUNCATE TABLE Staging.Pedidos;

-- 2. Load from source
INSERT INTO Staging.Pedidos (pedido_id, cliente_id, ...)
SELECT * FROM SourceDB.dbo.Pedidos;

-- 3. Validate & Transform
INSERT INTO DW.Fact_Pedidos (...)
SELECT 
    pedido_id,
    cliente_id,
    CONVERT(DATE, data_pedido, 103) AS data_pedido,  -- Corrigir formato
    CAST(REPLACE(REPLACE(valor_total, '$', ''), ',', '') AS DECIMAL(10,2)) AS valor_total,
    UPPER(TRIM(status)) AS status
FROM Staging.Pedidos
WHERE TRY_CONVERT(DATE, data_pedido, 103) IS NOT NULL  -- Validação
  AND TRY_CAST(REPLACE(REPLACE(valor_total, '$', ''), ',', '') AS DECIMAL(10,2)) IS NOT NULL;
```

**Quando usar**: Sempre em processos ETL complexos. Permite debugging e reprocessamento sem re-extrair da fonte.

**Relacionado**: ETL, Data Quality, Error Handling

---

### **Full Load**
**Carga Completa**

**Definição**: Estratégia de extração que copia TODOS os dados da fonte para o destino, independentemente de terem mudado ou não.

**Vantagens**:
- Simples de implementar
- Garante consistência total
- Não precisa de controle de mudanças

**Desvantagens**:
- Lento para tabelas grandes
- Consome muita banda
- Sobrecarrega sistema fonte

**Exemplo**:
```sql
-- Truncar destino
TRUNCATE TABLE DW.Dim_Produtos;

-- Carregar tudo
INSERT INTO DW.Dim_Produtos (produto_id, nome, categoria, ...)
SELECT produto_id, nome, categoria, ...
FROM OLTP.dbo.Produtos;
```

**Quando usar**: Para tabelas pequenas (< 1 milhão de linhas), dimensões que mudam pouco, ou quando histórico não importa (SCD Tipo 1).

**Relacionado**: Incremental Load, SCD, ETL

---

### **Incremental Load**
**Carga Incremental**

**Definição**: Estratégia de extração que copia apenas os dados novos ou modificados desde a última carga, usando um ponto de referência (timestamp, sequence, etc.).

**Métodos**:
1. **Timestamp**: `WHERE modified_date > @last_load`
2. **High Water Mark**: `WHERE id > @last_id`
3. **CDC**: Captura de mudanças via logs

**Exemplo**:
```sql
-- Controle de carga
CREATE TABLE ETL_Control (
    table_name VARCHAR(100) PRIMARY KEY,
    last_load_timestamp DATETIME2,
    last_id BIGINT
);

-- Carga incremental por timestamp
DECLARE @last_load DATETIME2;
SELECT @last_load = last_load_timestamp 
FROM ETL_Control 
WHERE table_name = 'Pedidos';

INSERT INTO DW.Fact_Pedidos (...)
SELECT ...
FROM OLTP.dbo.Pedidos
WHERE modified_date > @last_load;

-- Atualizar controle
UPDATE ETL_Control
SET last_load_timestamp = GETDATE()
WHERE table_name = 'Pedidos';
```

**Quando usar**: Para tabelas grandes e/ou com alto volume de mudanças. Essencial para cargas frequentes (hora em hora, tempo real).

**Relacionado**: Full Load, CDC, Upsert

---

### **Upsert**
**Update + Insert**

**Definição**: Operação que insere um registro se ele não existir, ou atualiza se já existir. Combina INSERT e UPDATE em uma única operação.

**Implementações**:

**SQL Server (MERGE)**:
```sql
MERGE DW.Dim_Cliente AS target
USING Staging.Cliente AS source
ON target.cliente_id = source.cliente_id
WHEN MATCHED THEN
    UPDATE SET 
        nome = source.nome,
        email = source.email,
        data_atualizacao = GETDATE()
WHEN NOT MATCHED THEN
    INSERT (cliente_id, nome, email, data_criacao)
    VALUES (source.cliente_id, source.nome, source.email, GETDATE());
```

**Alternativa (sem MERGE)**:
```sql
-- Update existentes
UPDATE t
SET 
    t.nome = s.nome,
    t.email = s.email
FROM DW.Dim_Cliente t
JOIN Staging.Cliente s ON t.cliente_id = s.cliente_id;

-- Insert novos
INSERT INTO DW.Dim_Cliente (cliente_id, nome, email)
SELECT cliente_id, nome, email
FROM Staging.Cliente s
WHERE NOT EXISTS (
    SELECT 1 FROM DW.Dim_Cliente t 
    WHERE t.cliente_id = s.cliente_id
);
```

**Quando usar**: Em cargas incrementais quando você não sabe se o registro já existe. Muito comum em ETL.

**Relacionado**: Incremental Load, ETL, Idempotência

---

## 2. Modelagem de Dados {#modelagem-de-dados}

### **Star Schema**
**Esquema Estrela**

**Definição**: Modelo dimensional onde uma tabela fato central se conecta diretamente a múltiplas tabelas dimensão, formando visualmente uma estrela.

**Estrutura**:
```
        Dim_Tempo
             |
Dim_Cliente - Fact_Vendas - Dim_Produto
             |
        Dim_Loja
```

**Características**:
- **Simples**: Fácil de entender e consultar
- **Desnormalizado**: Dimensões contêm redundância
- **Performance**: Menos JOINs = queries mais rápidas

**Exemplo Completo**:
```sql
-- Tabela Fato (centro da estrela)
CREATE TABLE Fact_Vendas (
    venda_id BIGINT PRIMARY KEY,
    data_key INT,           -- FK para Dim_Data
    cliente_key INT,        -- FK para Dim_Cliente
    produto_key INT,        -- FK para Dim_Produto
    loja_key INT,           -- FK para Dim_Loja
    quantidade INT,
    valor_unitario DECIMAL(10,2),
    valor_total DECIMAL(10,2),
    custo DECIMAL(10,2)
);

-- Dimensão Produto (pontas da estrela)
CREATE TABLE Dim_Produto (
    produto_key INT PRIMARY KEY,
    produto_id INT,
    sku VARCHAR(50),
    nome_produto VARCHAR(200),
    categoria VARCHAR(100),
    subcategoria VARCHAR(100),
    marca VARCHAR(100),
    cor VARCHAR(50)
    -- Tudo junto, mesmo que categoria se repita
);

-- Query simples com Star Schema
SELECT 
    p.categoria,
    d.ano,
    d.mes,
    SUM(f.valor_total) AS receita
FROM Fact_Vendas f
JOIN Dim_Produto p ON f.produto_key = p.produto_key
JOIN Dim_Data d ON f.data_key = d.data_key
GROUP BY p.categoria, d.ano, d.mes;
```

**Quando usar**: Para a maioria dos Data Warehouses. Balance entre simplicidade e performance.

**Relacionado**: Snowflake Schema, Dimensional Modeling, Kimball Methodology

---

### **Snowflake Schema**
**Esquema Floco de Neve**

**Definição**: Variação do Star Schema onde as dimensões são normalizadas em subdimensões, formando uma estrutura mais complexa.

**Estrutura**:
```
                    Dim_Tempo
                        |
Dim_Categoria - Dim_Produto
                        |
Dim_Cliente - Fact_Vendas - Dim_Produto - Dim_Marca
                        |
                   Dim_Loja - Dim_Regiao
```

**Exemplo**:
```sql
-- Star Schema teria tudo em Dim_Produto:
CREATE TABLE Dim_Produto_Star (
    produto_key INT PRIMARY KEY,
    nome VARCHAR(200),
    categoria VARCHAR(100),        -- Redundante
    subcategoria VARCHAR(100),     -- Redundante
    marca VARCHAR(100)             -- Redundante
);

-- Snowflake normaliza:
CREATE TABLE Dim_Produto_Snow (
    produto_key INT PRIMARY KEY,
    nome VARCHAR(200),
    subcategoria_key INT  -- FK
);

CREATE TABLE Dim_Subcategoria (
    subcategoria_key INT PRIMARY KEY,
    nome_subcategoria VARCHAR(100),
    categoria_key INT  -- FK
);

CREATE TABLE Dim_Categoria (
    categoria_key INT PRIMARY KEY,
    nome_categoria VARCHAR(100)
);
```

**Vantagens**:
- Reduz redundância
- Facilita manutenção de hierarquias

**Desvantagens**:
- Queries mais complexas (mais JOINs)
- Performance pior que Star Schema

**Quando usar**: Quando economia de storage é crítica ou hierarquias mudam frequentemente. Raro na prática.

**Relacionado**: Star Schema, Normalização, Dimensional Modeling

---

### **Fact Table**
**Tabela Fato**

**Definição**: Tabela central em modelo dimensional que armazena medidas quantitativas (métricas) de eventos de negócio, junto com chaves estrangeiras para dimensões.

**Tipos de Fatos**:

**1. Transaction Fact (Fato Transacional)**:
- Granularidade: Uma linha por transação
- Exemplo: Cada venda individual

```sql
CREATE TABLE Fact_Vendas_Transacional (
    venda_id BIGINT PRIMARY KEY,
    data_key INT,
    cliente_key INT,
    produto_key INT,
    quantidade INT,            -- Medida aditiva
    valor_total DECIMAL(10,2), -- Medida aditiva
    desconto_pct DECIMAL(5,2)  -- Medida não-aditiva
);
```

**2. Periodic Snapshot (Snapshot Periódico)**:
- Granularidade: Uma linha por período
- Exemplo: Saldo de conta bancária ao final de cada dia

```sql
CREATE TABLE Fact_Saldo_Diario (
    conta_key INT,
    data_key INT,
    saldo_final DECIMAL(15,2),    -- Semi-aditivo (não soma por tempo)
    saldo_medio DECIMAL(15,2),
    transacoes_dia INT,
    PRIMARY KEY (conta_key, data_key)
);
```

**3. Accumulating Snapshot (Snapshot Acumulativo)**:
- Granularidade: Uma linha por processo de negócio
- Exemplo: Pedido desde criação até entrega

```sql
CREATE TABLE Fact_Pedido_Lifecycle (
    pedido_id BIGINT PRIMARY KEY,
    data_criacao_key INT,
    data_pagamento_key INT,
    data_envio_key INT,
    data_entrega_key INT,
    tempo_pagamento_dias INT,   -- Calculado
    tempo_entrega_dias INT      -- Calculado
);
```

**Medidas - Tipos de Aditividade**:

1. **Aditiva**: Soma em todas as dimensões
   - Exemplo: quantidade vendida, receita

2. **Semi-aditiva**: Soma em algumas dimensões, NÃO em tempo
   - Exemplo: saldo de conta (não faz sentido somar saldo de jan + fev)

3. **Não-aditiva**: Não pode somar
   - Exemplo: percentuais, taxas, médias

**Quando usar**: Centro do seu modelo dimensional. Escolha o tipo baseado na natureza do evento de negócio.

**Relacionado**: Dimension Table, Star Schema, Grain

---

### **Dimension Table**
**Tabela Dimensão**

**Definição**: Tabela que contém atributos descritivos (contexto) usados para filtrar, agrupar e rotular medidas da tabela fato.

**Características**:
- **Desnormalizada**: Contém hierarquias completas
- **Descritiva**: Textos legíveis (não códigos)
- **Relativamente pequena**: Comparada às tabelas fato
- **Muda lentamente**: SCD (Slowly Changing Dimensions)

**Exemplo Completo**:
```sql
CREATE TABLE Dim_Produto (
    produto_key INT PRIMARY KEY,          -- Surrogate Key
    produto_id INT,                       -- Natural Key (ID no sistema fonte)
    sku VARCHAR(50),                      -- Código único
    nome_produto VARCHAR(200),
    descricao TEXT,
    -- Hierarquia de categorização
    categoria VARCHAR(100),
    subcategoria VARCHAR(100),
    departamento VARCHAR(100),
    -- Outros atributos
    marca VARCHAR(100),
    cor VARCHAR(50),
    tamanho VARCHAR(20),
    peso_kg DECIMAL(8,2),
    -- Controle SCD Tipo 2
    data_inicio DATE,
    data_fim DATE,
    is_current BIT,
    versao INT
);
```

**Tipos Especiais de Dimensões**:

**1. Junk Dimension (Dimensão Lixo)**:
Agrupa flags e indicadores binários para evitar explodir a fato.

```sql
-- ❌ Sem Junk Dimension (tabela fato enorme)
CREATE TABLE Fact_Vendas_Ruim (
    venda_id BIGINT,
    is_promocao BIT,
    is_fidelidade BIT,
    is_primeira_compra BIT,
    tipo_pagamento VARCHAR(20)
    -- Combinações = 2³ × n = muitas combinações vazias
);

-- ✅ Com Junk Dimension
CREATE TABLE Dim_Venda_Flags (
    flag_key INT PRIMARY KEY,
    is_promocao BIT,
    is_fidelidade BIT,
    is_primeira_compra BIT,
    tipo_pagamento VARCHAR(20)
);
-- Apenas 16 combinações reais (2⁴)

CREATE TABLE Fact_Vendas_Boa (
    venda_id BIGINT,
    flag_key INT  -- FK
);
```

**2. Degenerate Dimension (Dimensão Degenerada)**:
Atributo dimensional que fica na tabela fato (sem tabela dimensão própria).

```sql
CREATE TABLE Fact_Vendas (
    venda_id BIGINT,
    nota_fiscal VARCHAR(50),  -- Degenerate dimension
    cliente_key INT,
    produto_key INT
);
```

**3. Role-Playing Dimension (Dimensão Papel)**:
Mesma dimensão usada múltiplas vezes com papéis diferentes.

```sql
CREATE TABLE Fact_Pedido (
    pedido_id BIGINT,
    data_pedido_key INT,    -- FK para Dim_Data
    data_envio_key INT,     -- FK para Dim_Data (mesma tabela!)
    data_entrega_key INT    -- FK para Dim_Data (mesma tabela!)
);
```

**Quando usar**: Para TODOS os atributos descritivos. Regra de ouro: se você faz GROUP BY ou WHERE por algo, provavelmente é uma dimensão.

**Relacionado**: Fact Table, SCD, Surrogate Key

---

### **SCD (Slowly Changing Dimension)**
**Dimensão de Mudança Lenta**

**Definição**: Estratégias para lidar com mudanças em atributos dimensionais ao longo do tempo.

**Tipo 1: Sobrescrever (Sem Histórico)**

**Uso**: Correções ou atributos que não precisam de histórico.

```sql
-- Cliente muda de email
UPDATE Dim_Cliente
SET email = 'novo@email.com'
WHERE cliente_id = 123;

-- Histórico perdido!
```

**Tipo 2: Versionamento (Histórico Completo)**

**Uso**: Mudanças significativas onde histórico é crítico.

```sql
CREATE TABLE Dim_Cliente (
    cliente_key INT PRIMARY KEY IDENTITY,  -- Surrogate Key (nunca muda)
    cliente_id INT,                        -- Natural Key (pode ter múltiplas versões)
    nome VARCHAR(200),
    segmento VARCHAR(50),                  -- Ex: Bronze → Ouro
    -- Controle de versão
    data_inicio DATE,
    data_fim DATE,
    is_current BIT,
    versao INT
);

-- Exemplo de mudança de segmento
-- Versão 1 (antiga)
| cliente_key | cliente_id | segmento | data_inicio | data_fim   | is_current | versao |
|-------------|------------|----------|-------------|------------|------------|--------|
| 1001        | 123        | Bronze   | 2023-01-01  | 2024-03-15 | 0          | 1      |

-- Versão 2 (atual)
| cliente_key | cliente_id | segmento | data_inicio | data_fim   | is_current | versao |
|-------------|------------|----------|-------------|------------|------------|--------|
| 1002        | 123        | Ouro     | 2024-03-16  | 9999-12-31 | 1          | 2      |

-- Processo de mudança
BEGIN TRANSACTION;

-- Fechar versão antiga
UPDATE Dim_Cliente
SET data_fim = '2024-03-15', is_current = 0
WHERE cliente_id = 123 AND is_current = 1;

-- Criar nova versão
INSERT INTO Dim_Cliente (cliente_id, nome, segmento, data_inicio, data_fim, is_current, versao)
VALUES (123, 'João Silva', 'Ouro', '2024-03-16', '9999-12-31', 1, 2);

COMMIT;
```

**Benefício**: Queries históricas funcionam!
```sql
-- Vendas quando o cliente era Bronze
SELECT SUM(valor) 
FROM Fact_Vendas f
JOIN Dim_Cliente c ON f.cliente_key = c.cliente_key
WHERE c.segmento = 'Bronze';
```

**Tipo 3: Coluna Adicional (Histórico Limitado)**

**Uso**: Rastrear UMA mudança anterior.

```sql
CREATE TABLE Dim_Cliente (
    cliente_key INT PRIMARY KEY,
    cliente_id INT,
    segmento_atual VARCHAR(50),
    segmento_anterior VARCHAR(50),
    data_mudanca_segmento DATE
);

-- Mudança de Bronze para Ouro
UPDATE Dim_Cliente
SET 
    segmento_anterior = segmento_atual,
    segmento_atual = 'Ouro',
    data_mudanca_segmento = GETDATE()
WHERE cliente_id = 123;
```

**Comparação**:

| Tipo | Histórico | Complexidade | Storage | Uso Comum |
|------|-----------|--------------|---------|-----------|
| 1 | Nenhum | Baixa | Mínimo | Correções, dados não-críticos |
| 2 | Completo | Alta | Alto | Segmentação, preços, status |
| 3 | 1 mudança | Média | Médio | Upgrade de plano, mudança de região |

**Quando usar**:
- **Tipo 1**: Email, telefone (dados de contato)
- **Tipo 2**: Segmento de cliente, categoria de produto, endereço
- **Tipo 3**: Plano de assinatura (atual vs anterior)

**Relacionado**: Dimension Table, Data Warehouse, Historical Analysis

---

### **Surrogate Key**
**Chave Substituta**

**Definição**: Identificador único artificial (geralmente inteiro sequencial) usado como chave primária no Data Warehouse, em vez da chave natural do sistema fonte.

**Por que usar?**

**Problema com Natural Keys**:
```sql
-- Sistema fonte usa CPF como chave
CREATE TABLE OLTP.Clientes (
    cpf VARCHAR(14) PRIMARY KEY,  -- 123.456.789-00
    nome VARCHAR(200)
);

-- Problemas:
-- 1. CPF pode ter formato inconsistente (com/sem pontos)
-- 2. CPF pode mudar (retificação)
-- 3. String é mais lenta que INT em JOINs
-- 4. Ocupa mais espaço
-- 5. Dificulta SCD Tipo 2 (múltiplas versões)
```

**Solução com Surrogate Key**:
```sql
CREATE TABLE DW.Dim_Cliente (
    cliente_key INT PRIMARY KEY IDENTITY(1,1),  -- Surrogate Key
    cliente_id VARCHAR(14),                     -- Natural Key (CPF)
    nome VARCHAR(200),
    -- SCD Tipo 2
    data_inicio DATE,
    data_fim DATE,
    is_current BIT
);

-- Múltiplas versões do mesmo cliente natural
| cliente_key | cliente_id      | nome         | is_current |
|-------------|-----------------|--------------|------------|
| 1001        | 123.456.789-00  | João Silva   | 0          |
| 1002        | 123.456.789-00  | João Silva   | 1          |
```

**Vantagens**:
1. **Performance**: INT é mais rápido que VARCHAR em JOINs
2. **Independência**: Mudanças no sistema fonte não afetam DW
3. **SCD Tipo 2**: Permite múltiplas versões
4. **Simplicidade**: Sempre crescente, sem lógica de negócio

**Implementação**:
```sql
-- SQL Server
CREATE TABLE Dim_Produto (
    produto_key INT PRIMARY KEY IDENTITY(1,1),  -- Auto-incremento
    produto_id INT,
    nome VARCHAR(200)
);

-- Manual (em ETL)
DECLARE @next_key INT;
SELECT @next_key = ISNULL(MAX(produto_key), 0) + 1 FROM Dim_Produto;

INSERT INTO Dim_Produto (produto_key, produto_id, nome)
VALUES (@next_key, @source_id, @source_name);
```

**Quando usar**: Sempre em tabelas dimensão de Data Warehouse. Natural keys ficam como atributos.

**Relacionado**: Natural Key, Dimension Table, SCD

---

### **Grain (Granularidade)**
**Granularidade**

**Definição**: O nível mais atômico de detalhe representado em uma tabela fato. Define o que cada linha representa.

**Exemplos**:

**Granularidade Fina** (mais detalhado):
```sql
-- Grain: Uma linha por item de pedido
CREATE TABLE Fact_Vendas_Item (
    venda_item_id BIGINT PRIMARY KEY,
    pedido_id BIGINT,
    produto_key INT,
    data_key INT,
    quantidade INT,
    valor_unitario DECIMAL(10,2)
);
-- Grain: "Uma venda de UM produto específico em UM pedido"
```

**Granularidade Grossa** (mais agregado):
```sql
-- Grain: Uma linha por dia por loja
CREATE TABLE Fact_Vendas_Diario (
    loja_key INT,
    data_key INT,
    total_vendas INT,
    receita_total DECIMAL(15,2),
    PRIMARY KEY (loja_key, data_key)
);
-- Grain: "Total de vendas de UMA loja em UM dia"
```

**Regras de Ouro**:

1. **Definir ANTES de construir**: "Cada linha representa..."
2. **Consistente**: Todas as linhas devem ter o mesmo grain
3. **Mais fino possível**: Você pode agregar depois, mas não detalhar mais

**Exemplo de Grain Misto (ERRADO)**:
```sql
-- ❌ Grain inconsistente!
CREATE TABLE Fact_Vendas_Ruim (
    id BIGINT,
    nivel VARCHAR(20),  -- 'ITEM' ou 'PEDIDO'
    pedido_id BIGINT,
    produto_id INT,     -- NULL quando nivel='PEDIDO'
    valor DECIMAL(10,2)
);
-- Problema: Queries viram um pesadelo!
```

**Exemplo Correto (Grains Separados)**:
```sql
-- ✅ Grain 1: Item de pedido
CREATE TABLE Fact_Vendas_Item (
    venda_item_id BIGINT PRIMARY KEY,
    pedido_key INT,
    produto_key INT,
    quantidade INT,
    valor_total DECIMAL(10,2)
);

-- ✅ Grain 2: Pedido completo (agregado de itens)
CREATE TABLE Fact_Vendas_Pedido (
    pedido_key INT PRIMARY KEY,
    cliente_key INT,
    data_key INT,
    quantidade_itens INT,
    valor_total DECIMAL(10,2)
);
```

**Como escolher grain**:

Perguntas a fazer:
1. Qual a menor unidade de negócio que faz sentido analisar?
2. Quais perguntas de negócio preciso responder?
3. Qual o volume de dados?

**Exemplo E-commerce**:
- Pergunta: "Qual produto vende mais?" → Grain: Item
- Pergunta: "Qual ticket médio por pedido?" → Grain: Pedido
- Pergunta: "Qual receita diária?" → Grain: Dia

**Quando usar grains diferentes**:
- **Grain fino**: Para análises detalhadas, drill-down
- **Grain grosso**: Para dashboards executivos, séries temporais longas

**Relacionado**: Fact Table, Aggregation, Dimensional Modeling

---

### **Conformed Dimension**
**Dimensão Conformada**

**Definição**: Dimensão compartilhada por múltiplas tabelas fato, garantindo consistência de análises cruzadas.

**Problema sem Conformed Dimensions**:
```sql
-- Fact de Vendas tem sua própria Dim_Produto
CREATE TABLE Vendas_Dim_Produto (
    produto_key INT PRIMARY KEY,
    categoria VARCHAR(100)  -- "Eletrônicos"
);

-- Fact de Estoque tem OUTRA Dim_Produto
CREATE TABLE Estoque_Dim_Produto (
    produto_key INT PRIMARY KEY,
    categoria VARCHAR(100)  -- "Electronics" (diferente!)
);

-- Query cruzada falha
SELECT v.categoria, SUM(valor), AVG(estoque)
FROM Fact_Vendas v
JOIN Fact_Estoque e ON v.produto_id = e.produto_id  -- Categorias não batem!
```

**Solução com Conformed Dimension**:
```sql
-- Uma ÚNICA Dim_Produto compartilhada
CREATE TABLE Dim_Produto (
    produto_key INT PRIMARY KEY,
    produto_id INT,
    categoria VARCHAR(100)  -- Padronizado: "Eletrônicos"
);

-- Ambas as fatos apontam para a MESMA dimensão
CREATE TABLE Fact_Vendas (
    venda_id BIGINT,
    produto_key INT FOREIGN KEY REFERENCES Dim_Produto  -- Mesma!
);

CREATE TABLE Fact_Estoque (
    estoque_id BIGINT,
    produto_key INT FOREIGN KEY REFERENCES Dim_Produto  -- Mesma!
);

-- Query cruzada funciona
SELECT 
    p.categoria,
    SUM(v.valor_total) AS vendas,
    AVG(e.quantidade_estoque) AS estoque_medio
FROM Fact_Vendas v
JOIN Fact_Estoque e ON v.produto_key = e.produto_key
JOIN Dim_Produto p ON v.produto_key = p.produto_key
GROUP BY p.categoria;
```

**Exemplos típicos**:
- **Dim_Data**: Compartilhada por TODAS as fatos
- **Dim_Cliente**: Compartilhada por Vendas, Suporte, Marketing
- **Dim_Produto**: Compartilhada por Vendas, Estoque, Compras

**Benefícios**:
1. **Consistência**: Mesmas categorias, mesmas hierarquias
2. **Drill-across**: Analisar múltiplas fatos juntas
3. **Manutenção**: Atualiza em um lugar só

**Quando usar**: Sempre que a mesma entidade de negócio aparece em múltiplos processos.

**Relacionado**: Dimension Table, Enterprise Data Warehouse, Master Data Management

---

## 3. Estatística e Probabilidade {#estatistica-e-probabilidade}

### **Mean (Média)**
**Média Aritmética**

**Definição**: Soma de todos os valores dividida pela quantidade de observações.

**Fórmula**:
$$
\bar{x} = \frac{1}{n} \sum_{i=1}^{n} x_i
$$

**Exemplo**:
```python
import numpy as np

valores = [100, 150, 200, 250, 300]
media = np.mean(valores)  # 200

# Manualmente
media_manual = sum(valores) / len(valores)  # 200
```

**SQL**:
```sql
SELECT AVG(valor_pedido) AS ticket_medio
FROM Pedidos;
```

**Quando usar**: Para dados simétricos sem outliers extremos.

**Limitações**:
- **Sensível a outliers**: [10, 10, 10, 10, 1000] → média = 208 (não representativa!)
- **Não funciona para distribuições assimétricas**

**Relacionado**: Median, Mode, Outlier

---

### **Median (Mediana)**
**Mediana**

**Definição**: Valor central que divide a distribuição ao meio (50% abaixo, 50% acima).

**Fórmula** (dados ordenados):
$$
\text{Mediana} = 
\begin{cases}
x_{\frac{n+1}{2}} & \text{se } n \text{ ímpar} \\
\frac{x_{\frac{n}{2}} + x_{\frac{n}{2}+1}}{2} & \text{se } n \text{ par}
\end{cases}
$$

**Exemplo**:
```python
valores = [10, 10, 10, 10, 1000]
mediana = np.median(valores)  # 10 (muito melhor que média = 208!)

# Manualmente
valores_sorted = sorted(valores)
n = len(valores_sorted)
if n % 2 == 1:
    mediana = valores_sorted[n // 2]
else:
    mediana = (valores_sorted[n // 2 - 1] + valores_sorted[n // 2]) / 2
```

**SQL**:
```sql
-- SQL Server
SELECT 
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY valor_pedido) OVER () AS mediana
FROM Pedidos;
```

**Quando usar**:
- Dados com outliers (salários, preços imobiliários)
- Distribuições assimétricas
- Quando "valor típico" é mais importante que média matemática

**Relacionado**: Mean, Quartile, Robustness

---

### **Mode (Moda)**
**Moda**

**Definição**: Valor mais frequente na distribuição.

**Exemplo**:
```python
from scipy import stats

valores = [1, 2, 2, 3, 3, 3, 4, 5]
moda = stats.mode(valores)  # 3 (aparece 3 vezes)
```

**SQL**:
```sql
SELECT TOP 1 produto_id, COUNT(*) AS frequencia
FROM Vendas
GROUP BY produto_id
ORDER BY COUNT(*) DESC;
```

**Tipos**:
- **Unimodal**: Uma moda
- **Bimodal**: Duas modas
- **Multimodal**: Múltiplas modas

**Quando usar**: Para dados categóricos ou discretos (produto mais vendido, cor mais popular).

**Relacionado**: Mean, Median, Frequency Distribution

---

### **Variance (Variância)**
**Variância**

**Definição**: Média dos desvios quadrados em relação à média. Mede dispersão dos dados.

**Fórmula População**:
$$
\sigma^2 = \frac{1}{N} \sum_{i=1}^{N} (x_i - \mu)^2
$$

**Fórmula Amostra** (ddof=1):
$$
s^2 = \frac{1}{n-1} \sum_{i=1}^{n} (x_i - \bar{x})^2
$$

**Exemplo**:
```python
valores = [10, 12, 14, 16, 18]

# Variância populacional
var_pop = np.var(valores, ddof=0)  # 8.0

# Variância amostral (mais comum)
var_sample = np.var(valores, ddof=1)  # 10.0
```

**SQL**:
```sql
SELECT VAR(valor_pedido) AS variancia
FROM Pedidos;
```

**Interpretação**:
- **Variância baixa**: Dados concentrados perto da média
- **Variância alta**: Dados muito dispersos

**Problema**: Unidade é quadrada (ex: reais²), difícil interpretar.
**Solução**: Use Desvio Padrão (raiz da variância).

**Relacionado**: Standard Deviation, Dispersion, Spread

---

### **Standard Deviation (Desvio Padrão)**
**Desvio Padrão**

**Definição**: Raiz quadrada da variância. Mede dispersão na mesma unidade dos dados.

**Fórmula**:
$$
\sigma = \sqrt{\sigma^2} = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (x_i - \mu)^2}
$$

**Exemplo**:
```python
valores = [10, 12, 14, 16, 18]

desvio = np.std(valores, ddof=1)  # 3.16
media = np.mean(valores)  # 14

# Interpretação: Maioria dos dados está entre 14 ± 3.16 = [10.84, 17.16]
```

**SQL**:
```sql
SELECT 
    AVG(valor_pedido) AS media,
    STDEV(valor_pedido) AS desvio_padrao
FROM Pedidos;
```

**Regra 68-95-99.7 (para distribuição normal)**:
- 68% dos dados estão dentro de 1σ da média
- 95% dentro de 2σ
- 99.7% dentro de 3σ

**Quando usar**: Para entender se dados são "homogêneos" ou "heterogêneos".

**Relacionado**: Variance, Coefficient of Variation, Normal Distribution

---

### **Coefficient of Variation (CV)**
**Coeficiente de Variação**

**Definição**: Desvio padrão normalizado pela média. Permite comparar dispersões de datasets com escalas diferentes.

**Fórmula**:
$$
CV = \frac{\sigma}{\mu} \times 100\%
$$

**Exemplo**:
```python
# Dataset A: Salários
salarios = [2000, 2200, 2400, 2600, 2800]
cv_salarios = (np.std(salarios) / np.mean(salarios)) * 100  # 12.5%

# Dataset B: Preços de carros
carros = [20000, 22000, 24000, 26000, 28000]
cv_carros = (np.std(carros) / np.mean(carros)) * 100  # 12.5%

# Mesmo CV! Dispersão relativa é igual, embora valores absolutos sejam diferentes
```

**Interpretação**:
- **CV < 15%**: Baixa variabilidade (homogêneo)
- **15% < CV < 30%**: Variabilidade moderada
- **CV > 30%**: Alta variabilidade (heterogêneo)

**Quando usar**: Comparar dispersões de variáveis em escalas diferentes (salários vs preços, km/h vs m/s).

**Relacionado**: Standard Deviation, Relative Variability

---

### **Skewness (Assimetria)**
**Assimetria**

**Definição**: Medida da assimetria da distribuição em relação à média.

**Fórmula**:
$$
\text{Skewness} = \frac{E[(X - \mu)^3]}{\sigma^3}
$$

**Tipos**:

**1. Simétrica (Skew = 0)**:
```
      /\
     /  \
    /    \
   /      \
```
Média = Mediana = Moda

**2. Assimétrica à Direita (Skew > 0)**:
```
  /\
 /  \___
/       ---
```
Moda < Mediana < Média
Exemplo: Salários (poucos ganham muito)

**3. Assimétrica à Esquerda (Skew < 0)**:
```
       /\
  ____/  \
---       \
```
Média < Mediana < Moda
Exemplo: Notas de prova fácil (maioria tira nota alta)

**Exemplo**:
```python
from scipy.stats import skew

valores = [10, 20, 30, 40, 100]  # Outlier à direita
skewness = skew(valores)  # 1.35 (positivo = cauda à direita)

import matplotlib.pyplot as plt
plt.hist(valores, bins=10)
plt.title(f'Skewness: {skewness:.2f}')
plt.show()
```

**Interpretação**:
- **|Skew| < 0.5**: Aproximadamente simétrica
- **0.5 < |Skew| < 1**: Moderadamente assimétrica
- **|Skew| > 1**: Altamente assimétrica

**Quando importa**: 
- Escolha de medida central (média vs mediana)
- Teste de normalidade
- Transformações de dados (log, Box-Cox)

**Relacionado**: Kurtosis, Normal Distribution, Data Transformation

---

### **Kurtosis (Curtose)**
**Curtose**

**Definição**: Medida da "grossura das caudas" ou "pico" da distribuição.

**Fórmula**:
$$
\text{Kurtosis} = \frac{E[(X - \mu)^4]}{\sigma^4} - 3
$$
(Subtraímos 3 para que normal = 0)

**Tipos**:

**1. Mesocúrtica (Kurtosis = 0)**: Normal
```
     /\
    /  \
   /    \
  /      \
```

**2. Leptocúrtica (Kurtosis > 0)**: Pico alto, caudas pesadas
```
      /\
     /||\
    / || \
   /  ||  \
```
Mais outliers que o normal

**3. Platicúrtica (Kurtosis < 0)**: Pico baixo, caudas leves
```
    ____
   /    \
  /      \
 /        \
```
Menos outliers que o normal

**Exemplo**:
```python
from scipy.stats import kurtosis

# Distribuição com outliers extremos
valores = [10]*50 + [11]*50 + [100, 200, 300]  
kurt = kurtosis(valores)  # Positivo (caudas pesadas)

print(f"Kurtosis: {kurt:.2f}")
```

**Interpretação**:
- **Kurtosis > 3**: Cuidado com outliers!
- **Kurtosis < 0**: Distribuição "achatada", poucos extremos

**Quando usar**: Avaliar risco (finanças), detectar anomalias, validar suposição de normalidade.

**Relacionado**: Skewness, Outlier Detection, Fat Tails

---

### **Correlation (Correlação)**
**Correlação**

**Definição**: Medida de associação LINEAR entre duas variáveis.

**Pearson Correlation** (linear):
$$
r = \frac{\sum_{i=1}^{n}(x_i - \bar{x})(y_i - \bar{y})}{\sqrt{\sum_{i=1}^{n}(x_i - \bar{x})^2} \sqrt{\sum_{i=1}^{n}(y_i - \bar{y})^2}}
$$

Propriedades:
- $-1 \leq r \leq 1$
- $r = 1$: Correlação perfeita positiva
- $r = -1$: Correlação perfeita negativa
- $r = 0$: Sem correlação linear

**Exemplo**:
```python
import pandas as pd

df = pd.DataFrame({
    'tempo_site_min': [5, 10, 15, 20, 25],
    'valor_compra': [50, 75, 100, 125, 150]
})

# Pearson
corr_pearson = df['tempo_site_min'].corr(df['valor_compra'])  # 1.0 (perfeita!)

# Spearman (não-paramétrico, baseado em ranks)
from scipy.stats import spearmanr
corr_spearman, p_value = spearmanr(df['tempo_site_min'], df['valor_compra'])
```

**SQL (Pearson)**:
```sql
WITH Stats AS (
    SELECT 
        AVG(tempo_site) AS mean_x,
        AVG(valor_compra) AS mean_y,
        STDEV(tempo_site) AS std_x,
        STDEV(valor_compra) AS std_y
    FROM Sessoes
)
SELECT 
    SUM((s.tempo_site - st.mean_x) * (s.valor_compra - st.mean_y)) /
    (COUNT(*) * st.std_x * st.std_y) AS correlacao
FROM Sessoes s
CROSS JOIN Stats st;
```

**⚠️ ARMADILHA: Correlação ≠ Causalidade!**

```python
# Correlação espúria
vendas_sorvete = [100, 200, 300, 400, 500]
afogamentos = [10, 20, 30, 40, 50]

corr = np.corrcoef(vendas_sorvete, afogamentos)[0, 1]  # Alta correlação!
# Mas sorvete NÃO causa afogamento. Variável confundidora: temperatura (verão)
```

**Quando usar**: 
- Análise exploratória (heatmap de correlações)
- Feature selection em ML
- Validar relações esperadas

**Não usar para**:
- Relações não-lineares (use Spearman ou visualize scatter plot)
- Inferir causalidade (use experimentos, causal inference)

**Relacionado**: Covariance, Regression, Causation vs Correlation

---

### **Probability (Probabilidade)**
**Probabilidade**

**Definição**: Medida numérica da chance de um evento ocorrer.

**Axiomas de Kolmogorov**:
1. $P(A) \geq 0$ (não-negatividade)
2. $P(\Omega) = 1$ (normalização, onde $\Omega$ é o espaço amostral)
3. $P(A \cup B) = P(A) + P(B)$ se $A \cap B = \emptyset$ (aditividade)

**Exemplo**:
```python
# Experimento: Lançar dado
espaco_amostral = {1, 2, 3, 4, 5, 6}
evento_par = {2, 4, 6}

prob_par = len(evento_par) / len(espaco_amostral)  # 3/6 = 0.5
```

**Probabilidade Condicional**:
$$
P(A|B) = \frac{P(A \cap B)}{P(B)}
$$
"Probabilidade de A dado que B já ocorreu"

**Exemplo Fintech**:
```python
# P(Fraude | Transação Noturna)
total_transacoes = 10000
transacoes_noturnas = 2000
fraudes_noturnas = 100

prob_fraude_dada_noturna = fraudes_noturnas / transacoes_noturnas  # 0.05 = 5%
```

**Relacionado**: Bayes Theorem, Conditional Probability, Random Variable

---

### **Bayes Theorem**
**Teorema de Bayes**

**Definição**: Relaciona probabilidades condicionais inversas.

**Fórmula**:
$$
P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B)}
$$

Onde:
- $P(A|B)$: Probabilidade **posterior** (após observar B)
- $P(B|A)$: **Verossimilhança** (likelihood)
- $P(A)$: Probabilidade **prior** (antes de observar B)
- $P(B)$: **Evidência** (marginal)

**Exemplo Clássico: Teste de Fraude**

Dados:
- $P(\text{Fraude}) = 0.001$ (0.1% das transações)
- $P(\text{Alerta}|\text{Fraude}) = 0.95$ (sensibilidade 95%)
- $P(\text{Alerta}|\text{Legítima}) = 0.02$ (falso positivo 2%)

Pergunta: Se alerta dispara, qual P(Fraude | Alerta)?

```python
# Calcular P(Alerta)
p_fraude = 0.001
p_legitima = 1 - p_fraude
p_alerta_dado_fraude = 0.95
p_alerta_dado_legitima = 0.02

p_alerta = (p_alerta_dado_fraude * p_fraude) + (p_alerta_dado_legitima * p_legitima)
# = (0.95 × 0.001) + (0.02 × 0.999) = 0.02093

# Bayes
p_fraude_dado_alerta = (p_alerta_dado_fraude * p_fraude) / p_alerta
# = (0.95 × 0.001) / 0.02093 = 0.0454 = 4.54%

print(f"Mesmo com 95% de acurácia, apenas {p_fraude_dado_alerta:.2%} dos alertas são fraudes!")
```

**Lição**: **Base Rate Fallacy** - Não ignore a taxa base (prior)!

**Aplicações**:
- Sistemas antifraude
- Diagnóstico médico
- Spam filters
- Classificadores Bayesianos

**Relacionado**: Conditional Probability, Prior, Posterior, Likelihood

---

### **Normal Distribution (Distribuição Normal)**
**Distribuição Normal / Gaussiana**

**Definição**: Distribuição contínua simétrica em forma de sino, fundamental em estatística.

**PDF (Probability Density Function)**:
$$
f(x) = \frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{1}{2}\left(\frac{x-\mu}{\sigma}\right)^2}
$$

Parâmetros:
- $\mu$: Média (centro)
- $\sigma^2$: Variância (dispersão)

**Propriedades**:
1. Simétrica em torno de $\mu$
2. Média = Mediana = Moda = $\mu$
3. Regra 68-95-99.7:
   - 68% dos dados em $[\mu - \sigma, \mu + \sigma]$
   - 95% em $[\mu - 2\sigma, \mu + 2\sigma]$
   - 99.7% em $[\mu - 3\sigma, \mu + 3\sigma]$

**Exemplo**:
```python
from scipy.stats import norm
import matplotlib.pyplot as plt

mu = 100  # Média
sigma = 15  # Desvio padrão

# P(X < 110)
prob = norm.cdf(110, mu, sigma)  # 0.748

# P(90 < X < 110)
prob_intervalo = norm.cdf(110, mu, sigma) - norm.cdf(90, mu, sigma)  # 0.496

# Visualizar
x = np.linspace(mu - 4*sigma, mu + 4*sigma, 100)
y = norm.pdf(x, mu, sigma)

plt.plot(x, y)
plt.fill_between(x, y, where=(x >= 90) & (x <= 110), alpha=0.3)
plt.axvline(mu, color='r', linestyle='--', label='Média')
plt.title('Distribuição Normal (μ=100, σ=15)')
plt.show()
```

**Por que é importante?**
1. **Central Limit Theorem**: Médias amostrais são normais (independente da distribuição original)
2. Muitos fenômenos naturais são aproximadamente normais (altura, QI, erros de medição)
3. Fundamento de testes estatísticos (t-test, ANOVA, regressão)

**Quando usar**: Assumir normalidade para testes paramétricos, modelagem de variáveis contínuas.

**Relacionado**: CLT, Z-Score, Standard Normal, t-Distribution

---

### **Central Limit Theorem (CLT)**
**Teorema Central do Limite**

**Definição**: Para amostras grandes o suficiente, a distribuição das médias amostrais é aproximadamente normal, INDEPENDENTEMENTE da distribuição original.

**Fórmula**:
$$
\bar{X} \sim N\left(\mu, \frac{\sigma^2}{n}\right)
$$

Onde:
- $\bar{X}$: Média amostral
- $\mu$: Média populacional
- $\sigma^2$: Variância populacional
- $n$: Tamanho da amostra

**Regra prática**: $n \geq 30$ é suficiente para a maioria das distribuições.

**Demonstração**:
```python
from scipy.stats import expon
import matplotlib.pyplot as plt

# População: Exponencial (MUITO assimétrica)
populacao = expon(scale=2)

# Simular 10.000 médias amostrais (n=30)
medias_amostrais = []
for _ in range(10000):
    amostra = populacao.rvs(size=30)
    medias_amostrais.append(amostra.mean())

# Visualizar
fig, axes = plt.subplots(1, 2, figsize=(12, 4))

# Distribuição original (exponencial)
x = np.linspace(0, 10, 100)
axes[0].plot(x, populacao.pdf(x))
axes[0].set_title('Distribuição Original (Exponencial)')

# Distribuição das médias (normal pelo CLT!)
axes[1].hist(medias_amostrais, bins=50, density=True, alpha=0.7)
axes[1].set_title('Distribuição das Médias (Normal pelo CLT)')

plt.show()

# Confirmar normalidade
from scipy.stats import shapiro
stat, p_value = shapiro(medias_amostrais[:5000])  # Shapiro-Wilk test
print(f"p-value: {p_value:.4f}")  # p > 0.05 = normal!
```

**Por que é revolucionário?**
Permite usar testes estatísticos que ASSUMEM normalidade (t-test, ANOVA) mesmo quando dados originais NÃO são normais.

**Aplicações**:
- Intervalos de confiança
- Testes de hipóteses
- Controle de qualidade (cartas de controle)

**Relacionado**: Normal Distribution, Sampling Distribution, t-Distribution

---

### **Confidence Interval (IC)**
**Intervalo de Confiança**

**Definição**: Intervalo que contém o verdadeiro parâmetro populacional com certa probabilidade (confiança).

**Fórmula (média, variância conhecida)**:
$$
IC_{95\%} = \bar{x} \pm 1.96 \times \frac{\sigma}{\sqrt{n}}
$$

**Fórmula (média, variância desconhecida - mais comum)**:
$$
IC_{95\%} = \bar{x} \pm t_{\alpha/2, n-1} \times \frac{s}{\sqrt{n}}
$$

Onde:
- $\bar{x}$: Média amostral
- $t_{\alpha/2, n-1}$: Valor crítico da distribuição t (depende do nível de confiança e graus de liberdade)
- $s$: Desvio padrão amostral
- $n$: Tamanho da amostra

**Exemplo**:
```python
from scipy import stats
import numpy as np

# Dados: Ticket médio de uma amostra
amostra = [100, 120, 110, 130, 105, 115, 125, 108]
n = len(amostra)
media = np.mean(amostra)
desvio = np.std(amostra, ddof=1)  # ddof=1 para amostra

# IC 95%
confianca = 0.95
alpha = 1 - confianca
graus_liberdade = n - 1
t_critico = stats.t.ppf(1 - alpha/2, graus_liberdade)

margem_erro = t_critico * (desvio / np.sqrt(n))
ic_inferior = media - margem_erro
ic_superior = media + margem_erro

print(f"Média: R$ {media:.2f}")
print(f"IC 95%: [R$ {ic_inferior:.2f}, R$ {ic_superior:.2f}]")
print(f"Interpretação: Temos 95% de confiança que o ticket médio REAL está entre {ic_inferior:.2f} e {ic_superior:.2f}")
```

**⚠️ Interpretação CORRETA**:
"Se repetirmos esse experimento 100 vezes, ~95 dos intervalos conterão o verdadeiro parâmetro."

**❌ Interpretação ERRADA**:
"Há 95% de chance do verdadeiro parâmetro estar neste intervalo." (O parâmetro é fixo, não aleatório!)

**Fatores que afetam largura do IC**:
1. **Confiança**: 99% → IC mais largo que 95%
2. **Tamanho da amostra**: n maior → IC mais estreito
3. **Variabilidade**: Desvio maior → IC mais largo

**Quando usar**: Reportar estimativas com incerteza (surveys, A/B tests, forecasts).

**Relacionado**: Margin of Error, t-Distribution, Hypothesis Testing

---

### **Hypothesis Testing**
**Teste de Hipóteses**

**Definição**: Procedimento estatístico para decidir se uma afirmação sobre a população é plausível com base em evidências amostrais.

**Estrutura**:
1. **$H_0$ (Hipótese Nula)**: Status quo, "não há efeito"
2. **$H_1$ (Hipótese Alternativa)**: O que queremos provar
3. **$\alpha$ (Nível de significância)**: Probabilidade de Erro Tipo I (geralmente 0.05)
4. **p-valor**: Probabilidade de observar dados tão extremos quanto os observados, ASSUMINDO que $H_0$ é verdadeira
5. **Decisão**: 
   - Se p-valor < $\alpha$: Rejeitar $H_0$
   - Se p-valor ≥ $\alpha$: Não rejeitar $H_0$

**Erros**:
- **Tipo I**: Rejeitar $H_0$ quando é verdadeira (falso positivo) - Probabilidade = $\alpha$
- **Tipo II**: Não rejeitar $H_0$ quando é falsa (falso negativo) - Probabilidade = $\beta$

**Exemplo: t-test (comparar duas médias)**

```python
from scipy.stats import ttest_ind

# Grupo A: Conversão antes da mudança
conversoes_antes = [23, 25, 22, 24, 26, 21, 23, 25, 24, 22]

# Grupo B: Conversão depois da mudança
conversoes_depois = [28, 30, 27, 29, 31, 28, 30, 29, 28, 27]

# H0: Média(A) = Média(B) (mudança não teve efeito)
# H1: Média(A) ≠ Média(B) (mudança teve efeito)

t_stat, p_value = ttest_ind(conversoes_antes, conversoes_depois)

alpha = 0.05
print(f"Estatística t: {t_stat:.4f}")
print(f"P-valor: {p_value:.4f}")

if p_value < alpha:
    print(f"✅ Rejeitamos H0 (p = {p_value:.4f} < {alpha})")
    print("Conclusão: A mudança teve efeito significativo!")
else:
    print(f"❌ Não rejeitamos H0 (p = {p_value:.4f} >= {alpha})")
    print("Conclusão: Sem evidência de efeito significativo.")
```

**Tipos de Testes**:
- **t-test**: Comparar médias (1 ou 2 grupos)
- **ANOVA**: Comparar médias (3+ grupos)
- **Chi-squared**: Testar independência (dados categóricos)
- **Mann-Whitney**: Alternativa não-paramétrica ao t-test

**⚠️ ARMADILHA: P-hacking**
Se você testar 20 hipóteses com $\alpha = 0.05$, esperaria ~1 falso positivo por acaso!

**Correção de Bonferroni**:
$$
\alpha_{\text{ajustado}} = \frac{\alpha}{m}
$$
onde $m$ é o número de testes.

**Quando usar**: A/B tests, validar mudanças de produto, estudos científicos.

**Relacionado**: p-value, Statistical Significance, Power Analysis, Multiple Testing

---

### **p-value**
**Valor-p**

**Definição**: Probabilidade de observar dados tão ou mais extremos que os observados, ASSUMINDO que a hipótese nula ($H_0$) é verdadeira.

**NÃO é**:
- ❌ Probabilidade de $H_0$ ser verdadeira
- ❌ Probabilidade de erro
- ❌ Tamanho do efeito

**É**:
- ✅ Medida de **quão incompatíveis** os dados são com $H_0$

**Exemplo Visual**:

```python
from scipy.stats import norm
import matplotlib.pyplot as plt

# H0: Média = 100
# Observado: Média amostral = 110
# Desvio padrão conhecido = 15, n = 30

mu_h0 = 100
observado = 110
sigma = 15
n = 30
se = sigma / np.sqrt(n)  # Standard error

# Calcular z-score
z = (observado - mu_h0) / se  # 3.65

# p-valor (bilateral)
p_value = 2 * (1 - norm.cdf(abs(z)))  # 0.00026

# Visualizar
x = np.linspace(mu_h0 - 4*se, mu_h0 + 4*se, 1000)
y = norm.pdf(x, mu_h0, se)

plt.plot(x, y, label='Distribuição sob H0')
plt.fill_between(x, y, where=(x >= observado), alpha=0.3, color='red', label=f'p-valor = {p_value:.5f}')
plt.fill_between(x, y, where=(x <= 2*mu_h0 - observado), alpha=0.3, color='red')
plt.axvline(observado, color='r', linestyle='--', label='Valor observado')
plt.axvline(mu_h0, color='b', linestyle='--', label='H0: μ = 100')
plt.legend()
plt.title('p-valor: Área nas caudas')
plt.show()
```

**Interpretação**:
- **p < 0.001**: Evidência muito forte contra $H_0$
- **0.001 ≤ p < 0.01**: Evidência forte
- **0.01 ≤ p < 0.05**: Evidência moderada
- **p ≥ 0.05**: Evidência fraca ou inexistente

**Limite mágico 0.05?**
É convenção, não lei. Contexto importa:
- Medicina: use p < 0.01 (custo de erro alto)
- Exploratório: p < 0.10 pode ser ok

**Relacionado**: Hypothesis Testing, Statistical Significance, Alpha Level

---

### **Statistical Power**
**Poder Estatístico**

**Definição**: Probabilidade de detectar um efeito quando ele REALMENTE existe (1 - β, onde β é Erro Tipo II).

**Fórmula**:
$$
\text{Power} = 1 - \beta = P(\text{Rejeitar } H_0 \mid H_1 \text{ é verdadeira})
$$

**Fatores que afetam Power**:
1. **Tamanho da amostra (n)**: ↑ n → ↑ Power
2. **Tamanho do efeito**: Efeito maior → Mais fácil detectar
3. **Nível de significância (α)**: ↑ α → ↑ Power (mas ↑ falsos positivos)
4. **Variabilidade**: ↓ Variância → ↑ Power

**Exemplo: Calcular tamanho de amostra necessário**

```python
from statsmodels.stats.power import ttest_power

# Parâmetros
effect_size = 0.5  # Cohen's d (médio)
alpha = 0.05
power_desejado = 0.80  # 80% de chance de detectar efeito

# Calcular n necessário
from statsmodels.stats.power import tt_solve_power

n_necessario = tt_solve_power(
    effect_size=effect_size,
    alpha=alpha,
    power=power_desejado,
    alternative='two-sided'
)

print(f"Tamanho de amostra necessário: {n_necessario:.0f} por grupo")
```

**Regra de Ouro**: Aim for Power ≥ 0.80 (80%)

**Quando usar**: 
- **Antes** de experimento: Calcular tamanho de amostra
- **Depois** de experimento: Avaliar se tinha Power suficiente para detectar efeito

**Relacionado**: Sample Size, Effect Size, Type II Error

---

## 4. Análise Exploratória de Dados {#analise-exploratoria}

### **EDA (Exploratory Data Analysis)**
**Análise Exploratória de Dados**

**Definição**: Processo sistemático de investigar dados usando estatísticas descritivas e visualizações para descobrir padrões, anomalias e testar hipóteses iniciais.

**Objetivos**:
1. Entender distribuição das variáveis
2. Identificar outliers e anomalias
3. Detectar relações entre variáveis
4. Validar qualidade dos dados
5. Formular hipóteses para análise confirmatória

**Pipeline Típico**:
```
1. Load Data
   ↓
2. Summary Statistics (mean, median, std, min, max)
   ↓
3. Visualize Distributions (histogram, boxplot, QQ-plot)
   ↓
4. Detect Outliers (IQR, z-score)
   ↓
5. Analyze Relationships (correlation matrix, scatter plots)
   ↓
6. Handle Missing Data
   ↓
7. Document Findings
```

**Exemplo Completo**:
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# 1. Load
df = pd.read_csv('vendas.csv')

# 2. Summary Stats
print(df.describe())
print(df.info())
print(df.isnull().sum())

# 3. Distribuições
fig, axes = plt.subplots(2, 2, figsize=(12, 10))

# Histograma
df['valor_total'].hist(bins=50, ax=axes[0, 0])
axes[0, 0].set_title('Distribuição de Ticket Médio')

# Boxplot
df.boxplot(column='valor_total', by='categoria_produto', ax=axes[0, 1])

# QQ-plot (teste normalidade)
from scipy.stats import probplot
probplot(df['valor_total'], dist="norm", plot=axes[1, 0])

# KDE
df['valor_total'].plot(kind='kde', ax=axes[1, 1])
axes[1, 1].set_title('Densidade')

plt.tight_layout()
plt.show()

# 4. Outliers
Q1 = df['valor_total'].quantile(0.25)
Q3 = df['valor_total'].quantile(0.75)
IQR = Q3 - Q1
outliers = df[(df['valor_total'] < Q1 - 1.5*IQR) | (df['valor_total'] > Q3 + 1.5*IQR)]
print(f"Outliers: {len(outliers)} ({len(outliers)/len(df)*100:.2f}%)")

# 5. Correlações
corr_matrix = df[['valor_total', 'quantidade', 'desconto']].corr()
sns.heatmap(corr_matrix, annot=True, cmap='coolwarm')
plt.show()

# 6. Missing Data
print(df.isnull().sum())
# Decisão: Imputar, deletar ou flag?
```

**Quando usar**: SEMPRE antes de qualquer modelagem ou análise confirmatória.

**Relacionado**: Summary Statistics, Data Cleaning, Data Visualization

---

### **Outlier Detection**
**Detecção de Outliers**

**Definição**: Identificação de valores extremos que se desviam significativamente do padrão geral dos dados.

**Métodos**:

**1. IQR (Interquartile Range Method)**:
$$
\text{Outlier se } x < Q_1 - 1.5 \times IQR \text{ ou } x > Q_3 + 1.5 \times IQR
$$

```python
Q1 = df['valor'].quantile(0.25)
Q3 = df['valor'].quantile(0.75)
IQR = Q3 - Q1

limite_inferior = Q1 - 1.5 * IQR
limite_superior = Q3 + 1.5 * IQR

outliers = df[(df['valor'] < limite_inferior) | (df['valor'] > limite_superior)]
```

**2. Z-Score**:
$$
z = \frac{x - \mu}{\sigma}
$$
Outlier se $|z| > 3$ (ou 2.5 para mais sensibilidade)

```python
from scipy import stats

z_scores = np.abs(stats.zscore(df['valor']))
outliers = df[z_scores > 3]
```

**3. Isolation Forest (ML-based)**:
```python
from sklearn.ensemble import IsolationForest

clf = IsolationForest(contamination=0.05)  # Assumir 5% outliers
outlier_labels = clf.fit_predict(df[['valor', 'quantidade']])
outliers = df[outlier_labels == -1]
```

**O que fazer com outliers?**

1. **Investigar**: São erros ou legítimos?
   - Erro: `valor = -1000` → Deletar ou corrigir
   - Legítimo: Transação de R$ 1.000.000 em fintech → Manter!

2. **Opções**:
   - **Deletar**: Se claramente erro
   - **Winsorize**: Substituir por limite (percentil 99, por exemplo)
   - **Transformar**: Log, Box-Cox para reduzir impacto
   - **Flag**: Criar coluna `is_outlier` e tratar separadamente

```python
# Winsorize
from scipy.stats.mstats import winsorize
df['valor_winsorized'] = winsorize(df['valor'], limits=[0.01, 0.01])  # Trunque 1% em cada cauda

# Flag
df['is_outlier'] = (df['valor'] < limite_inferior) | (df['valor'] > limite_superior)
```

**Quando usar**: Sempre em EDA. Mas cuidado: nem todo outlier é erro!

**Relacionado**: IQR, Z-Score, Data Cleaning, Anomaly Detection

---

### **Missing Data**
**Dados Faltantes**

**Definição**: Valores ausentes em um dataset (NA, NULL, NaN).

**Tipos (taxonomia de Rubin)**:

**1. MCAR (Missing Completely At Random)**:
- Ausência **totalmente aleatória**, sem padrão
- Exemplo: Sensor quebra aleatoriamente
- **Teste**: Comparar médias entre observações com/sem missing
- **Solução**: Pode deletar (listwise deletion) ou imputar com média

**2. MAR (Missing At Random)**:
- Ausência **relacionada a outras variáveis**, mas não ao valor que faltaria
- Exemplo: Jovens não preenchem telefone fixo (relacionado a idade, não ao número em si)
- **Solução**: Imputação baseada em outras variáveis (regressão, KNN)

**3. MNAR (Missing Not At Random)**:
- Ausência **relacionada ao próprio valor que faltaria**
- Exemplo: Pessoas com renda baixa não declaram renda
- **Solução**: Modelagem complexa ou criar categoria "Não Informado"

**Estratégias**:

**1. Deletion**:
```python
# Listwise (deletar linha inteira)
df_clean = df.dropna()

# Pairwise (deletar apenas para análise específica)
corr = df[['var1', 'var2']].dropna().corr()
```

**2. Imputação Simples**:
```python
# Média
df['valor'].fillna(df['valor'].mean(), inplace=True)

# Mediana (mais robusta)
df['valor'].fillna(df['valor'].median(), inplace=True)

# Moda (categórica)
df['categoria'].fillna(df['categoria'].mode()[0], inplace=True)

# Forward/Backward Fill (séries temporais)
df['vendas'].fillna(method='ffill', inplace=True)  # Usar valor anterior
```

**3. Imputação Avançada**:
```python
# KNN Imputer
from sklearn.impute import KNNImputer

imputer = KNNImputer(n_neighbors=5)
df_imputed = pd.DataFrame(
    imputer.fit_transform(df[['var1', 'var2', 'var3']]),
    columns=['var1', 'var2', 'var3']
)

# MICE (Multiple Imputation by Chained Equations)
from sklearn.experimental import enable_iterative_imputer
from sklearn.impute import IterativeImputer

imputer = IterativeImputer(random_state=42)
df_imputed = pd.DataFrame(
    imputer.fit_transform(df),
    columns=df.columns
)
```

**4. Flag Missing**:
```python
df['valor_is_missing'] = df['valor'].isnull().astype(int)
df['valor'].fillna(df['valor'].median(), inplace=True)
# Modelo pode aprender que "missing" é informativo!
```

**Quando usar cada método**:
- **MCAR + <5% missing**: Delete
- **MAR**: Imputação baseada em outras variáveis
- **MNAR**: Flag + categoria "Unknown"
- **Séries temporais**: Forward/Backward fill

**Relacionado**: Data Cleaning, Imputation, Data Quality

---

### **Data Transformation**
**Transformação de Dados**

**Definição**: Aplicar funções matemáticas para modificar a escala ou distribuição dos dados.

**Objetivos**:
1. **Normalizar** distribuições (aproximar de normal)
2. **Estabilizar variância**
3. **Linearizar** relações
4. **Reduzir** impacto de outliers

**Tipos**:

**1. Log Transformation**:
$$
y = \log(x)
$$

Uso: Dados com cauda à direita (skew positivo)

```python
# Antes: Assimétrico
valores = [10, 20, 30, 100, 1000, 10000]

# Depois: Mais simétrico
valores_log = np.log(valores)

# Visualizar
fig, axes = plt.subplots(1, 2, figsize=(12, 4))
axes[0].hist(valores, bins=20)
axes[0].set_title('Original (Skew positivo)')

axes[1].hist(valores_log, bins=20)
axes[1].set_title('Log-transformado (Mais simétrico)')
plt.show()
```

**2. Square Root**:
$$
y = \sqrt{x}
$$

Uso: Dados de contagem (Poisson)

```python
contagens = [1, 4, 9, 16, 25, 100]
contagens_sqrt = np.sqrt(contagens)
```

**3. Box-Cox**:
$$
y(\lambda) = 
\begin{cases}
\frac{x^\lambda - 1}{\lambda} & \text{se } \lambda \neq 0 \\
\log(x) & \text{se } \lambda = 0
\end{cases}
$$

Uso: Encontra a melhor transformação automaticamente

```python
from scipy.stats import boxcox

valores = [10, 20, 30, 100, 1000, 10000]
valores_transformados, lambda_otimo = boxcox(valores)

print(f"Lambda ótimo: {lambda_otimo:.4f}")
# Se λ ≈ 0: Log
# Se λ ≈ 0.5: Square root
# Se λ ≈ 1: Sem transformação necessária
```

**4. Standardization (Z-score)**:
$$
z = \frac{x - \mu}{\sigma}
$$

Uso: Colocar variáveis em mesma escala

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
valores_padronizados = scaler.fit_transform(df[['valor1', 'valor2']])
# Média = 0, Desvio = 1
```

**5. Min-Max Normalization**:
$$
x_{\text{norm}} = \frac{x - x_{\min}}{x_{\max} - x_{\min}}
$$

Uso: Escalar para [0, 1]

```python
from sklearn.preprocessing import MinMaxScaler

scaler = MinMaxScaler()
valores_normalizados = scaler.fit_transform(df[['valor']])
# Range: [0, 1]
```

**Quando usar**:
- **Log**: Receita, preços, dados financeiros (skew positivo)
- **Box-Cox**: Quando não sabe qual transformação usar
- **Standardization**: Antes de algoritmos sensíveis a escala (KNN, SVM, Neural Networks)
- **Min-Max**: Quando precisa de range fixo [0, 1]

**Relacionado**: Normalization, Scaling, Distribution, Skewness

---

### **Feature Engineering**
**Engenharia de Features**

**Definição**: Criação de novas variáveis (features) a partir das existentes para melhorar performance de modelos ou facilitar análise.

**Técnicas**:

**1. Binning (Discretização)**:
```python
# Transformar variável contínua em categórica
df['faixa_idade'] = pd.cut(
    df['idade'], 
    bins=[0, 18, 35, 50, 100],
    labels=['Jovem', 'Adulto', 'Meia-idade', 'Idoso']
)

# Quantile-based
df['faixa_renda'] = pd.qcut(
    df['renda'],
    q=4,
    labels=['Baixa', 'Média-Baixa', 'Média-Alta', 'Alta']
)
```

**2. Interações**:
```python
# Multiplicação
df['receita_per_capita'] = df['receita_total'] / df['populacao']

# Razões
df['margem_lucro'] = (df['receita'] - df['custo']) / df['receita']

# Polinomiais
df['valor_squared'] = df['valor'] ** 2
```

**3. Agregações Temporais**:
```python
# Média móvel
df['vendas_ma_7d'] = df['vendas'].rolling(window=7).mean()

# Lag features
df['vendas_lag1'] = df['vendas'].shift(1)  # Valor de ontem
df['vendas_lag7'] = df['vendas'].shift(7)  # Valor 7 dias atrás

# Diff
df['vendas_diff'] = df['vendas'].diff()  # Mudança dia-a-dia
```

**4. Features de Data**:
```python
df['data'] = pd.to_datetime(df['data'])

df['ano'] = df['data'].dt.year
df['mes'] = df['data'].dt.month
df['dia_semana'] = df['data'].dt.dayofweek
df['dia_mes'] = df['data'].dt.day
df['trimestre'] = df['data'].dt.quarter
df['eh_fim_semana'] = df['dia_semana'].isin([5, 6]).astype(int)
df['eh_inicio_mes'] = (df['dia_mes'] <= 7).astype(int)
```

**5. Encoding Categórico**:
```python
# One-Hot Encoding
df_encoded = pd.get_dummies(df, columns=['categoria'], prefix='cat')

# Label Encoding (ordinais)
from sklearn.preprocessing import LabelEncoder
le = LabelEncoder()
df['tamanho_encoded'] = le.fit_transform(df['tamanho'])  # P, M, G → 0, 1, 2

# Target Encoding (usar com cuidado - pode vazar informação!)
mean_by_category = df.groupby('categoria')['target'].mean()
df['categoria_target_enc'] = df['categoria'].map(mean_by_category)
```

**6. Text Features**:
```python
# Comprimento
df['descricao_length'] = df['descricao'].str.len()

# Contagem de palavras
df['descricao_word_count'] = df['descricao'].str.split().str.len()

# TF-IDF
from sklearn.feature_extraction.text import TfidfVectorizer
tfidf = TfidfVectorizer(max_features=100)
tfidf_matrix = tfidf.fit_transform(df['descricao'])
```

**Exemplo E-commerce**:
```python
# RFM features
hoje = pd.Timestamp.now()
df['dias_desde_ultima_compra'] = (hoje - df['data_ultima_compra']).dt.days
df['frequencia_compra_mensal'] = df['total_pedidos'] / df['meses_desde_cadastro']
df['ticket_medio'] = df['valor_total_gasto'] / df['total_pedidos']

# Razões
df['pct_compras_com_desconto'] = df['compras_com_desconto'] / df['total_pedidos']
df['items_por_pedido'] = df['total_items'] / df['total_pedidos']

# Flags
df['eh_cliente_vip'] = (df['valor_total_gasto'] > df['valor_total_gasto'].quantile(0.9)).astype(int)
df['eh_cliente_em_risco'] = (df['dias_desde_ultima_compra'] > 180).astype(int)
```

**Quando usar**: Sempre! Boas features fazem mais diferença que algoritmos complexos.

**Relacionado**: Feature Selection, Dimensionality Reduction, Domain Knowledge

---

## 5. Métricas de Negócio {#metricas-de-negocio}

### **KPI (Key Performance Indicator)**
**Indicador-Chave de Performance**

**Definição**: Métrica quantificável que mede o sucesso em atingir objetivos de negócio.

**Características de um bom KPI**:
1. **SMART**:
   - **S**pecific: Claro e bem definido
   - **M**easurable: Quantificável
   - **A**chievable: Alcançável
   - **R**elevant: Alinhado com objetivos
   - **T**ime-bound: Com prazo

2. **Acionável**: Guia decisões

3. **Compreensível**: Stakeholders entendem

**Tipos**:

**Leading Indicators** (Preditivos):
- Medem atividades que CAUSAM resultados futuros
- Exemplo: Tráfego do site (prediz vendas futuras)

**Lagging Indicators** (Resultado):
- Medem resultados APÓS ações
- Exemplo: Receita do mês (resultado de ações passadas)

**Exemplo Framework - E-commerce**:

```
North Star Metric: Receita Mensal
    ↓
    = Visitantes × Taxa de Conversão × Ticket Médio
      ↓              ↓                   ↓
   [Tráfego]    [Conversão]        [Monetização]
```

**SQL para Calcular KPIs**:
```sql
WITH Metricas AS (
    SELECT 
        DATEFROMPARTS(YEAR(data_pedido), MONTH(data_pedido), 1) AS mes,
        COUNT(DISTINCT sessao_id) AS visitantes,
        COUNT(DISTINCT pedido_id) AS pedidos,
        SUM(valor_total) AS receita,
        SUM(valor_total) / NULLIF(COUNT(DISTINCT pedido_id), 0) AS ticket_medio
    FROM EventosSite
    WHERE data_pedido >= DATEADD(MONTH, -12, GETDATE())
    GROUP BY DATEFROMPARTS(YEAR(data_pedido), MONTH(data_pedido), 1)
)
SELECT 
    mes,
    visitantes,
    pedidos,
    receita,
    ticket_medio,
    CAST(pedidos AS FLOAT) / NULLIF(visitantes, 0) * 100 AS taxa_conversao_pct,
    -- Benchmark vs meta
    CASE 
        WHEN receita >= 1000000 THEN 'Acima da Meta'
        WHEN receita >= 800000 THEN 'No Track'
        ELSE 'Abaixo da Meta'
    END AS status
FROM Metricas
ORDER BY mes;
```

**Quando usar**: Para monitorar saúde do negócio e alinhar times.

**Relacionado**: North Star Metric, OKR, Dashboard

---

### **Conversion Rate (Taxa de Conversão)**
**Taxa de Conversão**

**Definição**: Percentual de visitantes/usuários que completam uma ação desejada.

**Fórmula Geral**:
$$
\text{Taxa de Conversão} = \frac{\text{Conversões}}{\text{Total de Visitantes}} \times 100\%
$$

**Variações por Setor**:

**E-commerce**:
```sql
-- Conversão de visitante para comprador
SELECT 
    COUNT(DISTINCT CASE WHEN comprou = 1 THEN sessao_id END) * 100.0 /
    NULLIF(COUNT(DISTINCT sessao_id), 0) AS taxa_conversao_checkout
FROM Sessoes
WHERE data_sessao >= DATEADD(DAY, -30, GETDATE());

-- Funil multi-etapa
WITH Funil AS (
    SELECT 
        SUM(CASE WHEN evento = 'visita' THEN 1 ELSE 0 END) AS etapa1_visita,
        SUM(CASE WHEN evento = 'produto_view' THEN 1 ELSE 0 END) AS etapa2_view,
        SUM(CASE WHEN evento = 'add_to_cart' THEN 1 ELSE 0 END) AS etapa3_cart,
        SUM(CASE WHEN evento = 'checkout' THEN 1 ELSE 0 END) AS etapa4_checkout,
        SUM(CASE WHEN evento = 'purchase' THEN 1 ELSE 0 END) AS etapa5_purchase
    FROM Eventos
)
SELECT 
    etapa1_visita,
    etapa2_view * 100.0 / NULLIF(etapa1_visita, 0) AS conv_visita_to_view,
    etapa3_cart * 100.0 / NULLIF(etapa2_view, 0) AS conv_view_to_cart,
    etapa4_checkout * 100.0 / NULLIF(etapa3_cart, 0) AS conv_cart_to_checkout,
    etapa5_purchase * 100.0 / NULLIF(etapa4_checkout, 0) AS conv_checkout_to_purchase,
    etapa5_purchase * 100.0 / NULLIF(etapa1_visita, 0) AS conv_overall
FROM Funil;
```

**SaaS (Trial to Paid)**:
```sql
SELECT 
    COUNT(DISTINCT CASE WHEN status = 'paid' THEN usuario_id END) * 100.0 /
    NULLIF(COUNT(DISTINCT CASE WHEN status IN ('trial', 'paid') THEN usuario_id END), 0) 
        AS taxa_conversao_trial_to_paid
FROM Assinaturas
WHERE data_inicio_trial >= DATEADD(DAY, -90, GETDATE());
```

**Benchmarks Típicos**:
- **E-commerce**: 1-3%
- **SaaS Trial-to-Paid**: 10-25%
- **Landing Page**: 5-15%

**Análise de Conversão**:
```python
import pandas as pd
from scipy.stats import chi2_contingency

# A/B Test de Conversão
df = pd.DataFrame({
    'Variante': ['A']*1000 + ['B']*1000,
    'Converteu': [1]*25 + [0]*975 + [1]*40 + [0]*960  # A: 2.5%, B: 4%
})

# Tabela de contingência
contingency = pd.crosstab(df['Variante'], df['Converteu'])
print(contingency)

# Teste Chi-squared
chi2, p_value, dof, expected = chi2_contingency(contingency)
print(f"p-value: {p_value:.4f}")

if p_value < 0.05:
    print("✅ Diferença significativa entre variantes!")
```

**Quando usar**: Para medir eficácia de funis, campanhas, features.

**Relacionado**: Funnel Analysis, A/B Testing, CRO (Conversion Rate Optimization)

---

### **CAC (Customer Acquisition Cost)**
**Custo de Aquisição de Cliente**

**Definição**: Custo total para adquirir um novo cliente.

**Fórmula**:
$$
CAC = \frac{\text{Custo Total de Marketing + Vendas}}{\text{Número de Clientes Adquiridos}}
$$

**Exemplo SQL**:
```sql
WITH CustosMarketing AS (
    SELECT 
        DATEFROMPARTS(YEAR(data_custo), MONTH(data_custo), 1) AS mes,
        SUM(valor_custo) AS custo_total
    FROM CustosMarketing
    WHERE data_custo >= DATEADD(MONTH, -12, GETDATE())
    GROUP BY DATEFROMPARTS(YEAR(data_custo), MONTH(data_custo), 1)
),
ClientesAdquiridos AS (
    SELECT 
        DATEFROMPARTS(YEAR(data_primeiro_pedido), MONTH(data_primeiro_pedido), 1) AS mes,
        COUNT(DISTINCT cliente_id) AS novos_clientes
    FROM Clientes
    WHERE data_primeiro_pedido >= DATEADD(MONTH, -12, GETDATE())
    GROUP BY DATEFROMPARTS(YEAR(data_primeiro_pedido), MONTH(data_primeiro_pedido), 1)
)
SELECT 
    c.mes,
    c.custo_total,
    a.novos_clientes,
    c.custo_total / NULLIF(a.novos_clientes, 0) AS cac,
    -- CAC por canal
    c.custo_total / NULLIF(a.novos_clientes, 0) AS cac_blended
FROM CustosMarketing c
JOIN ClientesAdquiridos a ON c.mes = a.mes
ORDER BY c.mes;
```

**Breakdown por Canal**:
```sql
SELECT 
    canal_aquisicao,
    SUM(custo_marketing) AS custo,
    COUNT(DISTINCT cliente_id) AS clientes,
    SUM(custo_marketing) / NULLIF(COUNT(DISTINCT cliente_id), 0) AS cac_por_canal
FROM AquisicoesClientes
WHERE data_aquisicao >= DATEADD(MONTH, -3, GETDATE())
GROUP BY canal_aquisicao
ORDER BY cac_por_canal;
```

**Métricas Relacionadas**:

**1. Payback Period** (Tempo para recuperar CAC):
$$
\text{Payback} = \frac{CAC}{\text{Receita Mensal por Cliente}}
$$

```sql
WITH Metricas AS (
    SELECT 
        canal_aquisicao,
        AVG(cac) AS cac_medio,
        AVG(receita_primeiro_mes) AS mrr_medio
    FROM ClientesMetricas
    GROUP BY canal_aquisicao
)
SELECT 
    canal_aquisicao,
    cac_medio,
    mrr_medio,
    cac_medio / NULLIF(mrr_medio, 0) AS payback_meses
FROM Metricas;
```

**2. CAC Ratio** (Eficiência de vendas):
$$
\text{CAC Ratio} = \frac{\text{Novo ARR Adicionado} \times \text{Margem Bruta}}{CAC}
$$

Benchmark: CAC Ratio > 3 é saudável

**Exemplo Python**:
```python
# Simular CAC por coorte
coortes = pd.DataFrame({
    'Coorte': pd.date_range('2024-01', periods=12, freq='MS'),
    'Custo_Marketing': np.random.randint(50000, 150000, 12),
    'Novos_Clientes': np.random.randint(200, 500, 12)
})

coortes['CAC'] = coortes['Custo_Marketing'] / coortes['Novos_Clientes']
coortes['CAC_MA3'] = coortes['CAC'].rolling(window=3).mean()  # Média móvel

# Visualizar
import matplotlib.pyplot as plt

plt.figure(figsize=(12, 6))
plt.plot(coortes['Coorte'], coortes['CAC'], marker='o', label='CAC Mensal')
plt.plot(coortes['Coorte'], coortes['CAC_MA3'], marker='s', label='CAC MA(3)')
plt.axhline(y=300, color='r', linestyle='--', label='Target CAC')
plt.title('CAC Over Time')
plt.xlabel('Coorte')
plt.ylabel('CAC (R$)')
plt.legend()
plt.grid(True, alpha=0.3)
plt.show()
```

**Quando monitorar**: Mensalmente. CAC crescente = problema de eficiência.

**Relacionado**: LTV, Unit Economics, Marketing ROI

---

### **LTV / CLV (Lifetime Value / Customer Lifetime Value)**
**Valor do Tempo de Vida do Cliente**

**Definição**: Receita total que um cliente gera durante todo seu relacionamento com a empresa.

**Fórmulas**:

**1. Histórico (Cohort-based - mais preciso)**:
$$
LTV = \sum_{t=0}^{T} \frac{\text{Receita}_t}{(1 + r)^t}
$$

onde $r$ é a taxa de desconto (opcional)

**2. Preditivo (Modelo simples)**:
$$
LTV = \frac{\text{Receita Média por Mês} \times \text{Margem Bruta}}{\text{Churn Rate Mensal}}
$$

**3. Preditivo (Modelo com retenção)**:
$$
LTV = \text{ARPU} \times \frac{1}{\text{Churn Rate}} \times \text{Margem Bruta}
$$

**Exemplo SQL (Histórico por Coorte)**:
```sql
WITH PrimeiroPedido AS (
    SELECT 
        cliente_id,
        MIN(data_pedido) AS primeira_compra,
        DATEFROMPARTS(YEAR(MIN(data_pedido)), MONTH(MIN(data_pedido)), 1) AS coorte_mes
    FROM Pedidos
    GROUP BY cliente_id
),
ReceitaPorMes AS (
    SELECT 
        pp.cliente_id,
        pp.coorte_mes,
        DATEDIFF(MONTH, pp.primeira_compra, p.data_pedido) AS mes_desde_primeira,
        SUM(p.valor_total) AS receita_mes
    FROM Pedidos p
    JOIN PrimeiroPedido pp ON p.cliente_id = pp.cliente_id
    GROUP BY pp.cliente_id, pp.coorte_mes, DATEDIFF(MONTH, pp.primeira_compra, p.data_pedido)
)
SELECT 
    coorte_mes,
    mes_desde_primeira,
    AVG(receita_mes) AS receita_media_mes,
    SUM(AVG(receita_mes)) OVER (
        PARTITION BY coorte_mes 
        ORDER BY mes_desde_primeira
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS ltv_cumulativo
FROM ReceitaPorMes
GROUP BY coorte_mes, mes_desde_primeira
ORDER BY coorte_mes, mes_desde_primeira;
```

**Exemplo Python (Preditivo)**:
```python
# Dados
arpu = 50  # Average Revenue Per User (mensal)
churn_rate_mensal = 0.05  # 5% ao mês
margem_bruta = 0.70  # 70%

# LTV
ltv = (arpu / churn_rate_mensal) * margem_bruta
# = (50 / 0.05) * 0.70 = 1000 * 0.70 = R$ 700

print(f"LTV Estimado: R$ {ltv:.2f}")

# Lifetime esperado em meses
lifetime_meses = 1 / churn_rate_mensal  # 20 meses
print(f"Lifetime Esperado: {lifetime_meses:.1f} meses")
```

**LTV com Análise de Sobrevivência**:
```python
from lifelines import KaplanMeierFitter
import pandas as pd

# Dados de clientes
df = pd.DataFrame({
    'cliente_id': range(1000),
    'tenure_meses': np.random.exponential(20, 1000),  # Média 20 meses
    'churned': np.random.binomial(1, 0.4, 1000),  # 40% churn
    'receita_mensal': np.random.normal(50, 10, 1000)
})

# Kaplan-Meier
kmf = KaplanMeierFitter()
kmf.fit(df['tenure_meses'], df['churned'])

# Probabilidade de sobrevivência em cada mês
survival_curve = kmf.survival_function_

# LTV considerando probabilidade de retenção
ltv_por_mes = []
for mes in range(1, 25):
    prob_sobreviver = kmf.survival_function_at_times(mes).values[0]
    receita_esperada = df['receita_mensal'].mean() * prob_sobreviver
    ltv_por_mes.append(receita_esperada)

ltv_total = sum(ltv_por_mes)
print(f"LTV Total (24 meses): R$ {ltv_total:.2f}")
```

**Relação LTV:CAC**:

$$
\text{LTV:CAC Ratio} = \frac{LTV}{CAC}
$$

**Benchmarks**:
- **LTV:CAC < 1**: Você perde dinheiro!
- **LTV:CAC = 3**: Saudável
- **LTV:CAC > 5**: Muito bom, mas pode estar sub-investindo em marketing

**Quando usar**: Para decisões de investimento em marketing, pricing, retenção.

**Relacionado**: CAC, Churn Rate, Cohort Analysis, Unit Economics

---

### **Churn Rate (Taxa de Cancelamento)**
**Taxa de Churn**

**Definição**: Percentual de clientes que cancelam/param de usar o produto em um período.

**Fórmulas**:

**Customer Churn Rate**:
$$
\text{Churn Rate} = \frac{\text{Clientes que Cancelaram no Período}}{\text{Clientes no Início do Período}} \times 100\%
$$

**Revenue Churn Rate** (mais importante para SaaS):
$$
\text{Revenue Churn} = \frac{\text{MRR Perdido no Mês}}{\text{MRR no Início do Mês}} \times 100\%
$$

**Exemplo SQL**:
```sql
-- Customer Churn (mensal)
WITH ClientesStatus AS (
    SELECT 
        DATEFROMPARTS(YEAR(data_evento), MONTH(data_evento), 1) AS mes,
        COUNT(DISTINCT CASE 
            WHEN tipo_evento = 'ativo_inicio_mes' THEN cliente_id 
        END) AS clientes_inicio,
        COUNT(DISTINCT CASE 
            WHEN tipo_evento = 'cancelamento' THEN cliente_id 
        END) AS clientes_cancelados
    FROM EventosAssinatura
    WHERE data_evento >= DATEADD(MONTH, -12, GETDATE())
    GROUP BY DATEFROMPARTS(YEAR(data_evento), MONTH(data_evento), 1)
)
SELECT 
    mes,
    clientes_inicio,
    clientes_cancelados,
    CAST(clientes_cancelados AS FLOAT) / NULLIF(clientes_inicio, 0) * 100 AS churn_rate_pct
FROM ClientesStatus
ORDER BY mes;

-- Revenue Churn
WITH MRR_Status AS (
    SELECT 
        DATEFROMPARTS(YEAR(data_evento), MONTH(data_evento), 1) AS mes,
        SUM(CASE 
            WHEN tipo_evento = 'inicio_mes' THEN mrr 
            ELSE 0 
        END) AS mrr_inicio,
        SUM(CASE 
            WHEN tipo_evento = 'cancelamento' THEN mrr 
            ELSE 0 
        END) AS mrr_perdido,
        SUM(CASE 
            WHEN tipo_evento = 'downgrade' THEN mrr_diferenca 
            ELSE 0 
        END) AS mrr_contraction,
        SUM(CASE 
            WHEN tipo_evento = 'upgrade' THEN mrr_diferenca 
            ELSE 0 
        END) AS mrr_expansion
    FROM MRR_Eventos
    GROUP BY DATEFROMPARTS(YEAR(data_evento), MONTH(data_evento), 1)
)
SELECT 
    mes,
    mrr_inicio,
    mrr_perdido,
    mrr_contraction,
    mrr_expansion,
    -- Gross Churn (sem considerar expansão)
    (mrr_perdido + mrr_contraction) / NULLIF(mrr_inicio, 0) * 100 AS gross_churn_rate_pct,
    -- Net Churn (considerando expansão)
    (mrr_perdido + mrr_contraction - mrr_expansion) / NULLIF(mrr_inicio, 0) * 100 AS net_churn_rate_pct
FROM MRR_Status
ORDER BY mes;
```

**Net Negative Churn** (Santo Graal do SaaS):
$$
\text{Net Churn} < 0
$$
Significa que expansão (upsells) > churn. Base cresce MESMO perdendo clientes!

**Análise de Churn - Sobrevivência**:
```python
from lifelines import KaplanMeierFitter
import matplotlib.pyplot as plt

# Dados
df = pd.DataFrame({
    'cliente_id': range(1000),
    'tenure_meses': np.random.exponential(15, 1000),
    'churned': np.random.binomial(1, 0.5, 1000),
    'segmento': np.random.choice(['Free', 'Basic', 'Pro'], 1000)
})

# Kaplan-Meier por segmento
fig, ax = plt.subplots(figsize=(12, 6))

for segmento in df['segmento'].unique():
    mask = df['segmento'] == segmento
    kmf = KaplanMeierFitter()
    kmf.fit(df[mask]['tenure_meses'], df[mask]['churned'], label=segmento)
    kmf.plot_survival_function(ax=ax)

plt.title('Curva de Retenção por Segmento')
plt.xlabel('Meses desde Signup')
plt.ylabel('Probabilidade de Retenção')
plt.grid(True, alpha=0.3)
plt.show()
```

**Churn Prediction (Modelo Preditivo)**:
```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report, roc_auc_score

# Features
features = [
    'dias_desde_ultima_atividade',
    'login_count_30d',
    'feature_usage_score',
    'support_tickets_count',
    'mrr',
    'tenure_meses'
]

X = df[features]
y = df['vai_churnar_proximo_mes']  # Target binário

# Split
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=42)

# Treinar
clf = RandomForestClassifier(n_estimators=100, random_state=42)
clf.fit(X_train, y_train)

# Predições
y_pred = clf.predict(X_test)
y_pred_proba = clf.predict_proba(X_test)[:, 1]

# Avaliar
print(classification_report(y_test, y_pred))
print(f"ROC AUC: {roc_auc_score(y_test, y_pred_proba):.4f}")

# Identificar clientes em risco
df['churn_probability'] = clf.predict_proba(X)[:, 1]
em_risco = df[df['churn_probability'] > 0.7].sort_values('churn_probability', ascending=False)
print(f"\nClientes em alto risco de churn: {len(em_risco)}")
```

**Benchmarks**:
- **SaaS B2B**: 5-7% anual
- **SaaS B2C**: 30-50% anual
- **E-commerce (repurchase)**: Varia muito

**Quando usar**: Monitorar mensalmente. Churn é o "vazamento" do balde de clientes.

**Relacionado**: Retention, LTV, Cohort Analysis, Survival Analysis

---

### **MRR / ARR (Monthly/Annual Recurring Revenue)**
**Receita Recorrente Mensal/Anual**

**Definição**: Receita previsível e recorrente de assinaturas.

**Fórmula**:
$$
MRR = \sum_{i=1}^{n} \text{Valor da Assinatura Mensal}_i
$$

$$
ARR = MRR \times 12
$$

**Componentes do MRR**:

1. **Novo MRR**: Novos clientes
2. **Expansion MRR**: Upgrades, cross-sell, upsell
3. **Contraction MRR**: Downgrades
4. **Churn MRR**: Cancelamentos

$$
\text{Net New MRR} = \text{Novo} + \text{Expansion} - \text{Contraction} - \text{Churn}
$$

**Exemplo SQL**:
```sql
WITH MRR_Breakdown AS (
    SELECT 
        DATEFROMPARTS(YEAR(data_evento), MONTH(data_evento), 1) AS mes,
        -- Novo MRR
        SUM(CASE 
            WHEN tipo_evento = 'nova_assinatura' THEN valor_mensal 
            ELSE 0 
        END) AS novo_mrr,
        -- Expansion MRR
        SUM(CASE 
            WHEN tipo_evento = 'upgrade' THEN valor_diferenca 
            ELSE 0 
        END) AS expansion_mrr,
        -- Contraction MRR
        SUM(CASE 
            WHEN tipo_evento = 'downgrade' THEN ABS(valor_diferenca) 
            ELSE 0 
        END) AS contraction_mrr,
        -- Churn MRR
        SUM(CASE 
            WHEN tipo_evento = 'cancelamento' THEN valor_mensal 
            ELSE 0 
        END) AS churn_mrr
    FROM EventosAssinatura
    WHERE data_evento >= DATEADD(MONTH, -12, GETDATE())
    GROUP BY DATEFROMPARTS(YEAR(data_evento), MONTH(data_evento), 1)
)
SELECT 
    mes,
    novo_mrr,
    expansion_mrr,
    contraction_mrr,
    churn_mrr,
    -- Net New MRR
    (novo_mrr + expansion_mrr - contraction_mrr - churn_mrr) AS net_new_mrr,
    -- MRR Cumulativo
    SUM(novo_mrr + expansion_mrr - contraction_mrr - churn_mrr) OVER (
        ORDER BY mes 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS mrr_total
FROM MRR_Breakdown
ORDER BY mes;
```

**Visualização Python**:
```python
import matplotlib.pyplot as plt

df_mrr = pd.read_sql(mrr_query, conn)

# Waterfall Chart
fig, ax = plt.subplots(figsize=(14, 8))

# Setup
meses = df_mrr['mes']
novo = df_mrr['novo_mrr']
expansion = df_mrr['expansion_mrr']
contraction = -df_mrr['contraction_mrr']
churn = -df_mrr['churn_mrr']

# Plot
ax.bar(meses, novo, label='Novo MRR', color='green')
ax.bar(meses, expansion, bottom=novo, label='Expansion', color='lightgreen')
ax.bar(meses, contraction, bottom=novo+expansion, label='Contraction', color='orange')
ax.bar(meses, churn, bottom=novo+expansion+contraction, label='Churn', color='red')

# Net line
net_new = novo + expansion + contraction + churn
ax.plot(meses, net_new.cumsum(), marker='o', color='blue', linewidth=2, label='MRR Total')

ax.set_title('MRR Breakdown')
ax.set_xlabel('Mês')
ax.set_ylabel('MRR (R$)')
ax.legend()
ax.grid(True, alpha=0.3)
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

**Métricas Derivadas**:

**1. Quick Ratio** (eficiência de crescimento):
$$
\text{Quick Ratio} = \frac{\text{Novo MRR} + \text{Expansion MRR}}{\text{Contraction MRR} + \text{Churn MRR}}
$$

Benchmark: > 4 é excelente

**2. Logo Retention** (% de clientes que renovam):
$$
\text{Logo Retention} = \frac{\text{Clientes Ativos Fim do Mês}}{\text{Clientes Ativos Início do Mês}}
$$

**3. Net Dollar Retention (NDR)**:
$$
NDR = \frac{\text{MRR Início} + \text{Expansion} - \text{Contraction} - \text{Churn}}{\text{MRR Início}}
$$

NDR > 100% = Mesmo perdendo clientes, base cresce!

**Quando usar**: Core metric para SaaS. Monitore semanalmente.

**Relacionado**: Churn, LTV, Unit Economics, SaaS Metrics

---

(Continua...)

**[NOTA: Documento excede limite de caracteres. Vou criar a segunda parte do dicionário com os tópicos restantes: Segmentação/ML, Visualização/BI, Ferramentas e Siglas.]**
