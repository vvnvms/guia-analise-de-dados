# O Guia Definitivo de Análise de Dados
## Do Estagiário ao Sênior: Engenharia, Matemática e Estratégia de Negócio

**Feito Com Inteligência Artificial**

---

## Índice

1. [Introdução: O Ecossistema de Analytics](#introducao)
2. [Fundamentos: De Onde Vêm os Dados](#fundamentos)
3. [ETL/ELT: A Engenharia por Trás dos Dados](#etl-elt)
4. [EDA: Exploratory Data Analysis](#eda)
5. [Fundamentos de Probabilidade e Estatística](#probabilidade-estatistica)
6. [Segmentação Avançada: RFM e Além](#segmentacao)
7. [KPIs: A Ponte Entre Dados e Negócio](#kpis)
8. [Dashboards: Storytelling com Dados](#dashboards)
9. [Casos Práticos Integrados](#casos-praticos)
10. [Armadilhas e Boas Práticas](#armadilhas)

---

## 1. Introdução: O Ecossistema de Analytics {#introducao}

### 1.1 A Jornada do Dado ao Insight

Imagine que você é um detetive investigando um caso complexo. Você não começa pela conclusão; você começa pelas **evidências** (dados brutos), organiza essas evidências em um **quadro de investigação** (data warehouse), busca **padrões** (EDA e estatística), formula **hipóteses** (segmentação e modelagem), e finalmente apresenta suas **conclusões** (dashboards e relatórios).

Este guia seguirá essa mesma jornada, mas com rigor técnico e matemático que transforma intuição em ciência.

**O Fluxo Fundamental:**

```
Fontes de Dados → ETL/ELT → Data Warehouse → EDA → Modelagem Estatística
                                                          ↓
Dashboards ← KPIs ← Segmentação (RFM, Cohort, etc.) ← Análise
```

### 1.2 Por Que Três Setores?

- **E-commerce**: Foco em comportamento transacional (RFM, funis de conversão, cart abandonment)
- **Fintech/Banking**: Foco em risco, fraude e análise temporal (time series, detecção de anomalias)
- **SaaS**: Foco em métricas de produto (churn, activation, lifetime value)

Cada setor tem suas peculiaridades, mas os **fundamentos matemáticos e de engenharia são universais**.

---

## 2. Fundamentos: De Onde Vêm os Dados {#fundamentos}

### 2.1 Fontes de Dados Primárias

#### E-commerce
```sql
-- Estrutura típica de uma tabela de pedidos
CREATE TABLE Pedidos (
    pedido_id INT PRIMARY KEY,
    cliente_id INT,
    data_pedido DATETIME,
    valor_total DECIMAL(10,2),
    status_pedido VARCHAR(50),
    canal_venda VARCHAR(50)
);

CREATE TABLE ItensPedido (
    item_id INT PRIMARY KEY,
    pedido_id INT,
    produto_id INT,
    quantidade INT,
    preco_unitario DECIMAL(10,2),
    desconto_percentual DECIMAL(5,2)
);
```

#### Fintech
```sql
CREATE TABLE Transacoes (
    transacao_id BIGINT PRIMARY KEY,
    conta_origem_id INT,
    conta_destino_id INT,
    data_hora DATETIME2,
    valor DECIMAL(15,2),
    tipo_transacao VARCHAR(50), -- PIX, TED, Boleto, etc.
    status VARCHAR(20),
    localizacao_geo GEOGRAPHY
);
```

#### SaaS
```sql
CREATE TABLE EventosProduto (
    evento_id BIGINT PRIMARY KEY,
    usuario_id INT,
    timestamp_evento DATETIME2,
    tipo_evento VARCHAR(100), -- login, feature_use, upgrade, etc.
    propriedades NVARCHAR(MAX), -- JSON com metadados
    sessao_id VARCHAR(100)
);
```

### 2.2 A Anatomia dos Dados: OLTP vs OLAP

**OLTP (Online Transaction Processing)**:
- **Propósito**: Operações do dia a dia (inserir pedidos, registrar transações)
- **Normalização**: Alta (3NF ou superior) para evitar redundância
- **Queries**: INSERT, UPDATE, DELETE frequentes

**OLAP (Online Analytical Processing)**:
- **Propósito**: Análise histórica e agregações
- **Modelagem**: Star Schema ou Snowflake Schema (desnormalizado)
- **Queries**: SELECT com JOINs complexos e agregações pesadas

**Por que isso importa?**
Se você tentar fazer análises complexas diretamente no OLTP, vai:
1. **Travar o sistema de produção** (queries analíticas são pesadas)
2. **Ter performance ruim** (normalização prejudica agregações)
3. **Não ter histórico adequado** (OLTP costuma ter apenas snapshot atual)

**Exemplo prático:**

```sql
-- ❌ ERRADO: Query analítica em OLTP
SELECT 
    c.cliente_id,
    COUNT(p.pedido_id) AS total_pedidos,
    SUM(p.valor_total) AS valor_total_gasto
FROM Clientes c
LEFT JOIN Pedidos p ON c.cliente_id = p.cliente_id
WHERE p.data_pedido >= DATEADD(YEAR, -1, GETDATE())
GROUP BY c.cliente_id;
-- Problema: Vai fazer full table scan em produção!

-- ✅ CORRETO: Query em Data Warehouse (modelo dimensional)
SELECT 
    dc.cliente_id,
    SUM(fp.quantidade_pedidos) AS total_pedidos,
    SUM(fp.valor_total) AS valor_total_gasto
FROM DimCliente dc
LEFT JOIN FatoPedidos fp ON dc.cliente_key = fp.cliente_key
WHERE fp.data_key >= 20240101 -- Formato YYYYMMDD
GROUP BY dc.cliente_id;
-- Vantagem: Dados pré-agregados, índices otimizados, sem impacto em produção
```

---

## 3. ETL/ELT: A Engenharia por Trás dos Dados {#etl-elt}

### 3.1 ETL vs ELT: A Evolução

**ETL (Extract, Transform, Load)**:
```
Fonte → [Transformação fora do DW] → Data Warehouse
```

**ELT (Extract, Load, Transform)**:
```
Fonte → Data Warehouse → [Transformação dentro do DW]
```

**Por que ELT ganhou força?**
- **Cloud computing**: Processamento massivo ficou mais barato que movimento de dados
- **SQL moderno**: T-SQL, PostgreSQL, BigQuery são poderosos o suficiente para transformações complexas

### 3.2 Extract: Capturando Dados

#### Estratégias de Extração

**Full Load** (Carga Completa):
```sql
-- Simples mas ineficiente para tabelas grandes
INSERT INTO Staging.Pedidos
SELECT * FROM Producao.dbo.Pedidos;
```

**Incremental Load** (Carga Incremental):
```sql
-- Usando campo de timestamp
DECLARE @UltimaExtracao DATETIME2;
SELECT @UltimaExtracao = MAX(data_atualizacao) 
FROM DW.dbo.FatoPedidos;

INSERT INTO Staging.Pedidos
SELECT * 
FROM Producao.dbo.Pedidos
WHERE data_atualizacao > @UltimaExtracao;
```

**Change Data Capture (CDC)**:
```sql
-- Habilitar CDC no SQL Server
EXEC sys.sp_cdc_enable_table
    @source_schema = 'dbo',
    @source_name = 'Pedidos',
    @role_name = NULL;

-- Capturar mudanças
SELECT *
FROM cdc.dbo_Pedidos_CT
WHERE __$start_lsn > @last_lsn;
```

### 3.3 Transform: A Matemática da Limpeza

#### Problema 1: Outliers e Valores Absurdos

**Cenário E-commerce**: Pedido com valor R$ 999.999,99

**Detecção Estatística:**

Usamos o **Método IQR (Interquartile Range)**:

$$
IQR = Q_3 - Q_1
$$

$$
\text{Outlier se } x < Q_1 - 1.5 \times IQR \text{ ou } x > Q_3 + 1.5 \times IQR
$$

```sql
WITH Quartis AS (
    SELECT 
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY valor_total) 
            OVER () AS Q1,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY valor_total) 
            OVER () AS Q3,
        valor_total,
        pedido_id
    FROM Pedidos
),
Limites AS (
    SELECT 
        Q1,
        Q3,
        Q3 - Q1 AS IQR,
        Q1 - 1.5 * (Q3 - Q1) AS limite_inferior,
        Q3 + 1.5 * (Q3 - Q1) AS limite_superior
    FROM Quartis
)
SELECT p.*
FROM Pedidos p
CROSS JOIN Limites l
WHERE p.valor_total NOT BETWEEN l.limite_inferior AND l.limite_superior;
```

**Quando NÃO remover outliers:**
- **Fintech**: Transação de R$ 1.000.000 pode ser legítima (transferência empresarial)
- **Solução**: Criar flag `is_outlier` em vez de deletar

```sql
ALTER TABLE Pedidos ADD is_outlier BIT DEFAULT 0;

UPDATE p
SET p.is_outlier = 1
FROM Pedidos p
CROSS JOIN (
    SELECT 
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY valor_total) OVER () AS Q1,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY valor_total) OVER () AS Q3
    FROM Pedidos
) q
WHERE p.valor_total > q.Q3 + 1.5 * (q.Q3 - q.Q1);
```

#### Problema 2: Dados Faltantes (Missing Data)

**Tipos de Missing Data (MCAR, MAR, MNAR):**

1. **MCAR (Missing Completely At Random)**: Ausência aleatória, sem padrão
   - Exemplo: Sensor falha aleatoriamente
   - **Solução**: Pode deletar ou imputar com média

2. **MAR (Missing At Random)**: Ausência relacionada a outras variáveis
   - Exemplo: Usuários jovens não preenchem telefone
   - **Solução**: Imputação com modelos preditivos

3. **MNAR (Missing Not At Random)**: Ausência relacionada ao valor que faltaria
   - Exemplo: Pessoas com renda baixa não declaram renda
   - **Solução**: Modelagem complexa ou criar categoria "Não Informado"

**Imputação com SQL:**

```sql
-- Média simples (MCAR)
UPDATE Pedidos
SET desconto_aplicado = (SELECT AVG(desconto_aplicado) FROM Pedidos WHERE desconto_aplicado IS NOT NULL)
WHERE desconto_aplicado IS NULL;

-- Mediana (mais robusta a outliers)
WITH Mediana AS (
    SELECT 
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY desconto_aplicado) AS valor_mediana
    FROM Pedidos
    WHERE desconto_aplicado IS NOT NULL
)
UPDATE Pedidos
SET desconto_aplicado = (SELECT valor_mediana FROM Mediana)
WHERE desconto_aplicado IS NULL;

-- Imputação por grupo (MAR)
WITH MediaPorCategoria AS (
    SELECT 
        categoria_produto,
        AVG(preco_unitario) AS preco_medio
    FROM Produtos
    WHERE preco_unitario IS NOT NULL
    GROUP BY categoria_produto
)
UPDATE p
SET p.preco_unitario = m.preco_medio
FROM Produtos p
JOIN MediaPorCategoria m ON p.categoria_produto = m.categoria_produto
WHERE p.preco_unitario IS NULL;
```

### 3.4 Load: Estratégias de Carga

#### Slowly Changing Dimensions (SCD)

**Tipo 1**: Sobrescrever (perde histórico)
```sql
UPDATE DimCliente
SET email = @novo_email
WHERE cliente_id = @id;
```

**Tipo 2**: Versionamento (mantém histórico completo)
```sql
-- Marcar registro antigo como inativo
UPDATE DimCliente
SET data_fim = GETDATE(), is_current = 0
WHERE cliente_id = @id AND is_current = 1;

-- Inserir nova versão
INSERT INTO DimCliente (cliente_id, email, data_inicio, is_current)
VALUES (@id, @novo_email, GETDATE(), 1);
```

**Tipo 3**: Coluna adicional (histórico limitado)
```sql
ALTER TABLE DimCliente ADD email_anterior VARCHAR(255);

UPDATE DimCliente
SET email_anterior = email, email = @novo_email
WHERE cliente_id = @id;
```

**Quando usar cada tipo:**
- **Tipo 1**: Correções (typos) ou dados que não precisam de histórico (e.g., telefone)
- **Tipo 2**: Mudanças significativas (endereço, categoria de cliente, plano SaaS)
- **Tipo 3**: Uma única mudança importante (upgrade de plano, por exemplo)

---

## 4. EDA: Exploratory Data Analysis {#eda}

### 4.1 A Filosofia do EDA

EDA não é "brincar com os dados". É um processo **sistemático e estatisticamente fundamentado** para:
1. Entender a **distribuição** das variáveis
2. Identificar **relações** entre variáveis
3. Detectar **anomalias** e **padrões inesperados**
4. Formular **hipóteses** para análise confirmatória

### 4.2 Estatísticas Descritivas: Além da Média

**Cenário**: Análise de ticket médio em e-commerce

```python
import pandas as pd
import numpy as np
from scipy import stats
import matplotlib.pyplot as plt
import seaborn as sns

# Carregar dados
df = pd.read_sql("""
    SELECT valor_total 
    FROM Pedidos 
    WHERE data_pedido >= '2024-01-01'
""", connection)

# Estatísticas básicas
media = df['valor_total'].mean()
mediana = df['valor_total'].median()
desvio_padrao = df['valor_total'].std()
```

**Por que a média sozinha é perigosa?**

Considere dois cenários:

**Loja A**: [100, 100, 100, 100, 100] → média = 100
**Loja B**: [10, 10, 10, 10, 460] → média = 100

Mesma média, **realidades completamente diferentes**!

**Métricas de Dispersão:**

1. **Variância**:
$$
\sigma^2 = \frac{1}{N} \sum_{i=1}^{N} (x_i - \mu)^2
$$

2. **Desvio Padrão**:
$$
\sigma = \sqrt{\sigma^2}
$$

3. **Coeficiente de Variação** (normaliza dispersão pela média):
$$
CV = \frac{\sigma}{\mu} \times 100\%
$$

```python
cv = (desvio_padrao / media) * 100
print(f"Coeficiente de Variação: {cv:.2f}%")

# Se CV > 30%, a média é um péssimo representante do conjunto
if cv > 30:
    print("⚠️ Alta variabilidade! Use mediana e quartis.")
```

**Métricas de Forma:**

1. **Assimetria (Skewness)**:
$$
\text{Skewness} = \frac{E[(X - \mu)^3]}{\sigma^3}
$$

```python
skewness = df['valor_total'].skew()

if skewness > 1:
    print("Distribuição assimétrica à direita (cauda longa)")
elif skewness < -1:
    print("Distribuição assimétrica à esquerda")
else:
    print("Distribuição aproximadamente simétrica")
```

2. **Curtose (Kurtosis)**:
$$
\text{Kurtosis} = \frac{E[(X - \mu)^4]}{\sigma^4} - 3
$$

```python
kurtosis = df['valor_total'].kurtosis()

if kurtosis > 0:
    print("Distribuição leptocúrtica (picos altos, caudas pesadas)")
elif kurtosis < 0:
    print("Distribuição platicúrtica (picos baixos, caudas leves)")
```

### 4.3 Visualizações Fundamentais

#### Histograma vs KDE

```python
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# Histograma
axes[0].hist(df['valor_total'], bins=50, edgecolor='black', alpha=0.7)
axes[0].set_title('Distribuição de Ticket Médio - Histograma')
axes[0].set_xlabel('Valor (R$)')
axes[0].set_ylabel('Frequência')

# KDE (Kernel Density Estimation)
df['valor_total'].plot(kind='kde', ax=axes[1])
axes[1].set_title('Distribuição de Ticket Médio - KDE')
axes[1].set_xlabel('Valor (R$)')
axes[1].set_ylabel('Densidade')

plt.tight_layout()
plt.show()
```

**KDE é melhor que histograma?**
- **Histograma**: Mostra frequências reais, mas sensível ao número de bins
- **KDE**: Suaviza a distribuição, mas pode "inventar" densidades em regiões sem dados

**Use ambos!**

#### Box Plot: O Detector de Outliers Visual

```python
fig, ax = plt.subplots(figsize=(10, 6))
sns.boxplot(data=df, y='valor_total', ax=ax)
ax.set_title('Box Plot - Ticket Médio')
ax.set_ylabel('Valor (R$)')
plt.show()

# Calcular limites manualmente
Q1 = df['valor_total'].quantile(0.25)
Q3 = df['valor_total'].quantile(0.75)
IQR = Q3 - Q1
limite_superior = Q3 + 1.5 * IQR
limite_inferior = Q1 - 1.5 * IQR

outliers = df[(df['valor_total'] < limite_inferior) | 
               (df['valor_total'] > limite_superior)]
print(f"Outliers detectados: {len(outliers)}")
```

#### QQ-Plot: Teste Visual de Normalidade

```python
from scipy.stats import probplot

fig, ax = plt.subplots(figsize=(8, 6))
probplot(df['valor_total'], dist="norm", plot=ax)
ax.set_title('QQ-Plot - Teste de Normalidade')
plt.show()
```

**Interpretação:**
- Pontos próximos à linha diagonal → distribuição normal
- Desvios sistemáticos → distribuição não-normal

**Por que isso importa?**
Muitos testes estatísticos (t-test, ANOVA) **assumem normalidade**. Se seus dados não são normais, você precisa:
1. **Transformar** os dados (log, Box-Cox)
2. **Usar testes não-paramétricos** (Mann-Whitney, Kruskal-Wallis)

### 4.4 Análise Bivariada: Relações Entre Variáveis

#### Correlação: Pearson vs Spearman

**Pearson** (relações lineares):
$$
r = \frac{\sum_{i=1}^{n}(x_i - \bar{x})(y_i - \bar{y})}{\sqrt{\sum_{i=1}^{n}(x_i - \bar{x})^2} \sqrt{\sum_{i=1}^{n}(y_i - \bar{y})^2}}
$$

**Spearman** (relações monotônicas, mais robusto):
$$
\rho = 1 - \frac{6 \sum d_i^2}{n(n^2 - 1)}
$$
onde $d_i$ é a diferença entre os ranks.

```python
# Exemplo: Relação entre tempo no site e valor do pedido
df_analysis = pd.read_sql("""
    SELECT 
        p.valor_total,
        s.tempo_sessao_minutos,
        s.paginas_visitadas
    FROM Pedidos p
    JOIN Sessoes s ON p.sessao_id = s.sessao_id
""", connection)

# Matriz de correlação
correlacao_pearson = df_analysis.corr(method='pearson')
correlacao_spearman = df_analysis.corr(method='spearman')

print("Correlação de Pearson:")
print(correlacao_pearson)
print("\nCorrelação de Spearman:")
print(correlacao_spearman)

# Visualizar
sns.heatmap(correlacao_pearson, annot=True, cmap='coolwarm', center=0)
plt.title('Matriz de Correlação (Pearson)')
plt.show()
```

**⚠️ ARMADILHA CRÍTICA: Correlação ≠ Causalidade**

Se você encontrar correlação entre "sorvete vendido" e "afogamentos", isso **NÃO** significa que sorvete causa afogamentos. Ambos são causados por uma **variável confundidora**: temperatura (verão).

**Como testar causalidade:**
1. **Experimentos controlados** (A/B tests)
2. **Análise de séries temporais** (Granger causality)
3. **Modelos causais** (DAGs, propensity score matching)

---

## 5. Fundamentos de Probabilidade e Estatística {#probabilidade-estatistica}

### 5.1 Probabilidade: A Linguagem da Incerteza

#### Definição Axiomática (Kolmogorov)

Para um espaço amostral $\Omega$ e evento $A \subseteq \Omega$:

1. $P(A) \geq 0$ (não-negatividade)
2. $P(\Omega) = 1$ (normalização)
3. $P(A \cup B) = P(A) + P(B)$ se $A \cap B = \emptyset$ (aditividade)

**Exemplo Fintech**: Probabilidade de fraude em transação

```python
# Dados históricos
df_transacoes = pd.read_sql("""
    SELECT 
        CASE WHEN is_fraude = 1 THEN 'Fraude' ELSE 'Legítima' END AS tipo,
        COUNT(*) AS quantidade
    FROM Transacoes
    WHERE data_transacao >= DATEADD(MONTH, -6, GETDATE())
    GROUP BY is_fraude
""", connection)

total = df_transacoes['quantidade'].sum()
prob_fraude = df_transacoes[df_transacoes['tipo'] == 'Fraude']['quantidade'].values[0] / total

print(f"P(Fraude) = {prob_fraude:.4f}")
```

#### Probabilidade Condicional e Teorema de Bayes

$$
P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B)}
$$

**Cenário Real**: Sistema antifraude em Fintech

- $P(\text{Fraude})$ = 0.001 (0.1% das transações)
- $P(\text{Alerta}|\text{Fraude})$ = 0.95 (95% de sensibilidade)
- $P(\text{Alerta}|\text{Legítima})$ = 0.02 (2% de falso positivo)

**Pergunta**: Se o sistema dispara alerta, qual a probabilidade de ser fraude de verdade?

$$
P(\text{Fraude}|\text{Alerta}) = \frac{P(\text{Alerta}|\text{Fraude}) \cdot P(\text{Fraude})}{P(\text{Alerta})}
$$

$$
P(\text{Alerta}) = P(\text{Alerta}|\text{Fraude}) \cdot P(\text{Fraude}) + P(\text{Alerta}|\text{Legítima}) \cdot P(\text{Legítima})
$$

$$
P(\text{Alerta}) = 0.95 \times 0.001 + 0.02 \times 0.999 = 0.02093
$$

$$
P(\text{Fraude}|\text{Alerta}) = \frac{0.95 \times 0.001}{0.02093} = 0.0454 = 4.54\%
$$

**IMPACTO PRÁTICO**: Mesmo com 95% de acurácia, apenas **4.54% dos alertas são fraudes reais**! Isso é chamado de **base rate fallacy**.

```python
def bayes_fraud_probability(prior_fraud, sensitivity, false_positive_rate):
    """
    Calcula probabilidade posterior de fraude dado um alerta
    """
    # P(Alerta)
    p_alert = (sensitivity * prior_fraud) + (false_positive_rate * (1 - prior_fraud))
    
    # P(Fraude | Alerta)
    p_fraud_given_alert = (sensitivity * prior_fraud) / p_alert
    
    return p_fraud_given_alert

resultado = bayes_fraud_probability(0.001, 0.95, 0.02)
print(f"Probabilidade de fraude dado alerta: {resultado:.4f}")
```

### 5.2 Distribuições de Probabilidade

#### Distribuição Normal (Gaussiana)

$$
f(x) = \frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{1}{2}\left(\frac{x-\mu}{\sigma}\right)^2}
$$

**Propriedades:**
- Média = Mediana = Moda
- 68% dos dados dentro de 1σ
- 95% dos dados dentro de 1.96σ
- 99.7% dos dados dentro de 3σ (regra 68-95-99.7)

**Exemplo SaaS**: Tempo médio de sessão

```python
from scipy.stats import norm

# Parâmetros da distribuição
mu = 15  # minutos
sigma = 5

# Probabilidade de sessão > 20 minutos
prob = 1 - norm.cdf(20, mu, sigma)
print(f"P(Sessão > 20 min) = {prob:.4f}")

# Percentil 90 (90% das sessões são menores que...)
percentil_90 = norm.ppf(0.90, mu, sigma)
print(f"90% das sessões < {percentil_90:.2f} minutos")

# Visualizar
x = np.linspace(mu - 4*sigma, mu + 4*sigma, 100)
y = norm.pdf(x, mu, sigma)

plt.figure(figsize=(10, 6))
plt.plot(x, y, label=f'Normal({mu}, {sigma}²)')
plt.fill_between(x, y, where=(x > 20), alpha=0.3, label='P(X > 20)')
plt.axvline(mu, color='r', linestyle='--', label='Média')
plt.title('Distribuição de Tempo de Sessão')
plt.xlabel('Minutos')
plt.ylabel('Densidade')
plt.legend()
plt.show()
```

#### Distribuição de Poisson (Eventos Raros)

$$
P(X = k) = \frac{\lambda^k e^{-\lambda}}{k!}
$$

**Uso**: Número de transações por hora, bugs por deploy, churn por mês

**Exemplo E-commerce**: Pedidos por hora

```python
from scipy.stats import poisson

# Média de 12 pedidos/hora
lambda_param = 12

# P(receber exatamente 15 pedidos na próxima hora)
prob_15 = poisson.pmf(15, lambda_param)
print(f"P(X = 15) = {prob_15:.4f}")

# P(receber mais de 20 pedidos)
prob_mais_20 = 1 - poisson.cdf(20, lambda_param)
print(f"P(X > 20) = {prob_mais_20:.4f}")

# Visualizar
k = np.arange(0, 30)
pmf = poisson.pmf(k, lambda_param)

plt.figure(figsize=(10, 6))
plt.bar(k, pmf, alpha=0.7)
plt.axvline(lambda_param, color='r', linestyle='--', label=f'λ = {lambda_param}')
plt.title('Distribuição de Poisson - Pedidos/Hora')
plt.xlabel('Número de Pedidos')
plt.ylabel('Probabilidade')
plt.legend()
plt.show()
```

#### Distribuição Binomial (Sucessos em n Tentativas)

$$
P(X = k) = \binom{n}{k} p^k (1-p)^{n-k}
$$

**Exemplo SaaS**: Conversão de trial para pago

```python
from scipy.stats import binom

# 100 usuários em trial, taxa de conversão 30%
n = 100
p = 0.30

# P(converter exatamente 35 usuários)
prob_35 = binom.pmf(35, n, p)
print(f"P(X = 35) = {prob_35:.4f}")

# P(converter pelo menos 40)
prob_min_40 = 1 - binom.cdf(39, n, p)
print(f"P(X >= 40) = {prob_min_40:.4f}")

# Intervalo de confiança (onde esperamos 95% dos resultados)
lower = binom.ppf(0.025, n, p)
upper = binom.ppf(0.975, n, p)
print(f"95% dos resultados entre {lower} e {upper} conversões")
```

### 5.3 Inferência Estatística

#### Teorema Central do Limite

**O teorema mais importante da estatística:**

Para uma amostra de tamanho $n$ de qualquer distribuição com média $\mu$ e variância $\sigma^2$, a distribuição da **média amostral** aproxima-se de uma normal:

$$
\bar{X} \sim N\left(\mu, \frac{\sigma^2}{n}\right)
$$

**Por que isso é revolucionário?**
Mesmo que seus dados originais **não sejam normais**, a distribuição das médias **será normal** (se n for grande o suficiente, tipicamente n ≥ 30).

```python
# Demonstração do TCL com distribuição exponencial (muito assimétrica)
from scipy.stats import expon

# Distribuição original (exponencial)
lambda_exp = 0.5
original_dist = expon(scale=1/lambda_exp)

# Simular 10.000 médias amostrais com n=30
n_amostras = 10000
tamanho_amostra = 30
medias_amostrais = []

for _ in range(n_amostras):
    amostra = original_dist.rvs(size=tamanho_amostra)
    medias_amostrais.append(amostra.mean())

# Visualizar
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# Distribuição original
x = np.linspace(0, 10, 100)
axes[0].plot(x, original_dist.pdf(x))
axes[0].set_title('Distribuição Original (Exponencial)')
axes[0].set_xlabel('Valor')
axes[0].set_ylabel('Densidade')

# Distribuição das médias (normal pelo TCL)
axes[1].hist(medias_amostrais, bins=50, density=True, alpha=0.7, label='Médias Amostrais')
mu_teorico = 1/lambda_exp
sigma_teorico = (1/lambda_exp) / np.sqrt(tamanho_amostra)
x_normal = np.linspace(min(medias_amostrais), max(medias_amostrais), 100)
axes[1].plot(x_normal, norm.pdf(x_normal, mu_teorico, sigma_teorico), 'r-', label='Normal Teórica')
axes[1].set_title('Distribuição das Médias (Normal pelo TCL)')
axes[1].set_xlabel('Média Amostral')
axes[1].set_ylabel('Densidade')
axes[1].legend()

plt.tight_layout()
plt.show()
```

#### Intervalos de Confiança

**Definição**: Intervalo que tem X% de probabilidade de conter o verdadeiro parâmetro populacional.

$$
IC_{95\%} = \bar{x} \pm 1.96 \times \frac{\sigma}{\sqrt{n}}
$$

**Exemplo E-commerce**: Ticket médio com IC 95%

```python
from scipy.stats import t

# Dados
df_tickets = pd.read_sql("SELECT valor_total FROM Pedidos WHERE data_pedido >= '2024-01-01'", connection)
amostra = df_tickets['valor_total'].values

n = len(amostra)
media_amostral = np.mean(amostra)
desvio_amostral = np.std(amostra, ddof=1)  # ddof=1 para amostra (não população)
erro_padrao = desvio_amostral / np.sqrt(n)

# Usar distribuição t (mais conservadora para amostras menores)
confianca = 0.95
alpha = 1 - confianca
graus_liberdade = n - 1
t_critico = t.ppf(1 - alpha/2, graus_liberdade)

margem_erro = t_critico * erro_padrao
ic_inferior = media_amostral - margem_erro
ic_superior = media_amostral + margem_erro

print(f"Ticket Médio: R$ {media_amostral:.2f}")
print(f"IC 95%: [R$ {ic_inferior:.2f}, R$ {ic_superior:.2f}]")
print(f"Interpretação: Temos 95% de confiança que o ticket médio REAL está entre R$ {ic_inferior:.2f} e R$ {ic_superior:.2f}")
```

#### Testes de Hipóteses

**Estrutura:**
- $H_0$ (hipótese nula): Não há efeito/diferença
- $H_1$ (hipótese alternativa): Há efeito/diferença
- $\alpha$ (nível de significância): Probabilidade de rejeitar $H_0$ quando é verdadeira (Erro Tipo I)

**Exemplo Fintech**: Nova feature de antifraude reduziu fraudes?

```python
from scipy.stats import ttest_ind

# Dados antes e depois da implementação
fraudes_antes = pd.read_sql("""
    SELECT COUNT(*) AS fraudes
    FROM Transacoes
    WHERE is_fraude = 1 
      AND data_transacao BETWEEN '2024-01-01' AND '2024-01-31'
    GROUP BY CAST(data_transacao AS DATE)
""", connection)

fraudes_depois = pd.read_sql("""
    SELECT COUNT(*) AS fraudes
    FROM Transacoes
    WHERE is_fraude = 1 
      AND data_transacao BETWEEN '2024-02-01' AND '2024-02-28'
    GROUP BY CAST(data_transacao AS DATE)
""", connection)

# Teste t independente
t_stat, p_value = ttest_ind(fraudes_antes['fraudes'], fraudes_depois['fraudes'])

print(f"Estatística t: {t_stat:.4f}")
print(f"P-valor: {p_value:.4f}")

if p_value < 0.05:
    print("✅ Rejeitamos H0: A feature teve efeito significativo (p < 0.05)")
else:
    print("❌ Não rejeitamos H0: Sem evidência de efeito significativo")
```

**⚠️ ARMADILHA: P-hacking**

Se você testar 20 hipóteses com α=0.05, **esperaria 1 falso positivo por acaso**!

**Correção de Bonferroni:**
$$
\alpha_{\text{ajustado}} = \frac{\alpha}{m}
$$
onde $m$ é o número de testes.

```python
from scipy.stats import ttest_ind

# Múltiplas comparações (exemplo: testar 10 features)
features = ['feature_A', 'feature_B', 'feature_C', ...]  # 10 features
p_values = []

for feature in features:
    # Teste para cada feature
    t_stat, p_val = ttest_ind(antes[feature], depois[feature])
    p_values.append(p_val)

# Correção de Bonferroni
alpha = 0.05
alpha_bonferroni = alpha / len(features)

significativos = [p < alpha_bonferroni for p in p_values]
print(f"Features significativas após correção: {sum(significativos)}/{len(features)}")
```

---

## 6. Segmentação Avançada: RFM e Além {#segmentacao}

### 6.1 RFM: Recency, Frequency, Monetary

**Fundamento**: Clientes que compraram **recentemente**, **frequentemente** e gastaram **muito** têm maior valor.

#### Implementação Matemática

**Passo 1: Calcular RFM em SQL**

```sql
DECLARE @DataReferencia DATE = GETDATE();

WITH RFM_Calculado AS (
    SELECT 
        cliente_id,
        -- Recency: dias desde a última compra
        DATEDIFF(DAY, MAX(data_pedido), @DataReferencia) AS Recency,
        -- Frequency: número de pedidos
        COUNT(DISTINCT pedido_id) AS Frequency,
        -- Monetary: valor total gasto
        SUM(valor_total) AS Monetary
    FROM Pedidos
    WHERE data_pedido >= DATEADD(YEAR, -1, @DataReferencia)
    GROUP BY cliente_id
),
RFM_Scores AS (
    SELECT 
        cliente_id,
        Recency,
        Frequency,
        Monetary,
        -- Scores de 1 a 5 usando NTILE (quintis)
        NTILE(5) OVER (ORDER BY Recency DESC) AS R_Score,  -- DESC porque menor recency é melhor
        NTILE(5) OVER (ORDER BY Frequency) AS F_Score,
        NTILE(5) OVER (ORDER BY Monetary) AS M_Score
    FROM RFM_Calculado
)
SELECT 
    cliente_id,
    R_Score,
    F_Score,
    M_Score,
    CAST(R_Score AS VARCHAR) + CAST(F_Score AS VARCHAR) + CAST(M_Score AS VARCHAR) AS RFM_Score,
    -- Segmentação
    CASE 
        WHEN R_Score >= 4 AND F_Score >= 4 AND M_Score >= 4 THEN 'Champions'
        WHEN R_Score >= 3 AND F_Score >= 3 AND M_Score >= 3 THEN 'Loyal Customers'
        WHEN R_Score >= 4 AND F_Score <= 2 AND M_Score <= 2 THEN 'New Customers'
        WHEN R_Score <= 2 AND F_Score >= 3 AND M_Score >= 3 THEN 'At Risk'
        WHEN R_Score <= 2 AND F_Score <= 2 THEN 'Lost'
        ELSE 'Outros'
    END AS Segmento
FROM RFM_Scores;
```

**Passo 2: Análise Estatística dos Segmentos**

```python
# Carregar dados RFM
df_rfm = pd.read_sql("SELECT * FROM RFM_Scores", connection)

# Estatísticas por segmento
segmentos_stats = df_rfm.groupby('Segmento').agg({
    'cliente_id': 'count',
    'Recency': ['mean', 'median'],
    'Frequency': ['mean', 'median'],
    'Monetary': ['mean', 'median', 'sum']
}).round(2)

print(segmentos_stats)

# Teste ANOVA: Os segmentos são estatisticamente diferentes em Monetary?
from scipy.stats import f_oneway

champions = df_rfm[df_rfm['Segmento'] == 'Champions']['Monetary']
loyal = df_rfm[df_rfm['Segmento'] == 'Loyal Customers']['Monetary']
at_risk = df_rfm[df_rfm['Segmento'] == 'At Risk']['Monetary']

f_stat, p_value = f_oneway(champions, loyal, at_risk)
print(f"\nANOVA F-statistic: {f_stat:.4f}, p-value: {p_value:.4e}")

if p_value < 0.05:
    print("✅ Segmentos têm valores monetários significativamente diferentes")
```

**Passo 3: Visualização Avançada**

```python
import seaborn as sns

# Heatmap de segmentos RFM
pivot_table = df_rfm.pivot_table(
    values='cliente_id', 
    index='R_Score', 
    columns='F_Score', 
    aggfunc='count'
)

plt.figure(figsize=(10, 8))
sns.heatmap(pivot_table, annot=True, fmt='g', cmap='YlOrRd')
plt.title('Mapa de Calor RFM - Recency vs Frequency')
plt.xlabel('Frequency Score')
plt.ylabel('Recency Score')
plt.show()

# 3D scatter plot
from mpl_toolkits.mplot3d import Axes3D

fig = plt.figure(figsize=(12, 8))
ax = fig.add_subplot(111, projection='3d')

colors = {'Champions': 'gold', 'Loyal Customers': 'green', 'At Risk': 'red', 
          'New Customers': 'blue', 'Lost': 'gray', 'Outros': 'lightgray'}

for segmento, color in colors.items():
    subset = df_rfm[df_rfm['Segmento'] == segmento]
    ax.scatter(subset['Recency'], subset['Frequency'], subset['Monetary'], 
               c=color, label=segmento, alpha=0.6, s=50)

ax.set_xlabel('Recency (dias)')
ax.set_ylabel('Frequency (pedidos)')
ax.set_zlabel('Monetary (R$)')
ax.set_title('Segmentação RFM em 3D')
ax.legend()
plt.show()
```

### 6.2 Além do RFM: Técnicas Avançadas

#### Cohort Analysis (Análise de Coorte)

**Definição**: Agrupar usuários pelo período de aquisição e analisar comportamento ao longo do tempo.

```sql
WITH PrimeiroPedido AS (
    SELECT 
        cliente_id,
        MIN(DATEFROMPARTS(YEAR(data_pedido), MONTH(data_pedido), 1)) AS cohort_mes
    FROM Pedidos
    GROUP BY cliente_id
),
PedidosPorMes AS (
    SELECT 
        p.cliente_id,
        pp.cohort_mes,
        DATEFROMPARTS(YEAR(p.data_pedido), MONTH(p.data_pedido), 1) AS pedido_mes,
        SUM(p.valor_total) AS valor_total
    FROM Pedidos p
    JOIN PrimeiroPedido pp ON p.cliente_id = pp.cliente_id
    GROUP BY p.cliente_id, pp.cohort_mes, DATEFROMPARTS(YEAR(p.data_pedido), MONTH(p.data_pedido), 1)
)
SELECT 
    cohort_mes,
    DATEDIFF(MONTH, cohort_mes, pedido_mes) AS meses_desde_aquisicao,
    COUNT(DISTINCT cliente_id) AS clientes_ativos,
    SUM(valor_total) AS receita_total
FROM PedidosPorMes
GROUP BY cohort_mes, DATEDIFF(MONTH, cohort_mes, pedido_mes)
ORDER BY cohort_mes, meses_desde_aquisicao;
```

**Visualização em Python:**

```python
# Carregar dados de coorte
df_cohort = pd.read_sql(cohort_query, connection)

# Criar matriz de retenção
cohort_pivot = df_cohort.pivot_table(
    index='cohort_mes',
    columns='meses_desde_aquisicao',
    values='clientes_ativos'
)

# Calcular taxa de retenção (normalizar pela coorte inicial)
cohort_size = cohort_pivot.iloc[:, 0]
retention_matrix = cohort_pivot.divide(cohort_size, axis=0) * 100

# Visualizar
plt.figure(figsize=(14, 8))
sns.heatmap(retention_matrix, annot=True, fmt='.0f', cmap='RdYlGn', vmin=0, vmax=100)
plt.title('Taxa de Retenção por Coorte (%)')
plt.xlabel('Meses desde Aquisição')
plt.ylabel('Coorte de Aquisição')
plt.show()
```

#### K-Means Clustering (Segmentação Não-Supervisionada)

**Quando usar**: Quando você quer descobrir segmentos **naturais** nos dados, sem definir regras a priori.

```python
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler

# Preparar dados
features = df_rfm[['Recency', 'Frequency', 'Monetary']].values

# Normalizar (K-Means é sensível a escala)
scaler = StandardScaler()
features_scaled = scaler.fit_transform(features)

# Método do Cotovelo para escolher número de clusters
inertias = []
K_range = range(2, 11)

for k in K_range:
    kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
    kmeans.fit(features_scaled)
    inertias.append(kmeans.inertia_)

# Plotar
plt.figure(figsize=(10, 6))
plt.plot(K_range, inertias, 'bo-')
plt.xlabel('Número de Clusters (k)')
plt.ylabel('Inércia')
plt.title('Método do Cotovelo')
plt.xticks(K_range)
plt.grid(True)
plt.show()

# Escolher k=4 (exemplo)
k_optimal = 4
kmeans_final = KMeans(n_clusters=k_optimal, random_state=42, n_init=10)
df_rfm['Cluster_KMeans'] = kmeans_final.fit_predict(features_scaled)

# Analisar clusters
cluster_summary = df_rfm.groupby('Cluster_KMeans').agg({
    'Recency': 'mean',
    'Frequency': 'mean',
    'Monetary': 'mean',
    'cliente_id': 'count'
}).round(2)
cluster_summary.columns = ['Avg_Recency', 'Avg_Frequency', 'Avg_Monetary', 'Count']
print(cluster_summary)
```

**Comparação: RFM vs K-Means**

| Aspecto | RFM | K-Means |
|---------|-----|---------|
| **Interpretabilidade** | Alta (regras claras) | Média (depende de análise posterior) |
| **Flexibilidade** | Baixa (fixo em R, F, M) | Alta (pode usar mais features) |
| **Requer domínio de negócio** | Sim | Menos |
| **Escalabilidade** | Excelente (SQL simples) | Boa (mas requer processamento Python) |

**Quando usar cada um:**
- **RFM**: E-commerce tradicional, quando stakeholders precisam entender segmentos facilmente
- **K-Means**: SaaS com muitas métricas de engajamento, quando quer descobrir padrões ocultos

---

## 7. KPIs: A Ponte Entre Dados e Negócio {#kpis}

### 7.1 Taxonomia de KPIs

#### KPIs de Vaidade vs KPIs Acionáveis

**KPI de Vaidade**: Métrica que parece boa mas não guia decisões
- Exemplo: "10 milhões de pageviews"

**KPI Acionável**: Métrica que informa ação
- Exemplo: "Taxa de conversão de landing page caiu 15% → testar novo CTA"

#### Framework SMART para KPIs

- **S**pecific: "Aumentar vendas" ❌ → "Aumentar conversão de checkout" ✅
- **M**easurable: Tem métrica clara
- **A**chievable: Realista
- **R**elevant: Alinhado com objetivo de negócio
- **T**ime-bound: Prazo definido

### 7.2 KPIs por Setor

#### E-commerce

**1. Taxa de Conversão**

$$
\text{Conversão} = \frac{\text{Pedidos}}{\text{Visitantes}} \times 100\%
$$

```sql
WITH Metricas AS (
    SELECT 
        CAST(data_evento AS DATE) AS data,
        COUNT(DISTINCT CASE WHEN tipo_evento = 'visita' THEN sessao_id END) AS visitantes,
        COUNT(DISTINCT CASE WHEN tipo_evento = 'compra' THEN sessao_id END) AS compradores
    FROM EventosSite
    WHERE data_evento >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY CAST(data_evento AS DATE)
)
SELECT 
    data,
    visitantes,
    compradores,
    CAST(compradores AS FLOAT) / NULLIF(visitantes, 0) * 100 AS taxa_conversao_pct
FROM Metricas
ORDER BY data;
```

**2. Abandoned Cart Rate**

$$
\text{Abandonment} = \frac{\text{Carrinhos Criados} - \text{Carrinhos Convertidos}}{\text{Carrinhos Criados}} \times 100\%
$$

```sql
SELECT 
    COUNT(DISTINCT carrinho_id) AS carrinhos_criados,
    COUNT(DISTINCT CASE WHEN pedido_id IS NOT NULL THEN carrinho_id END) AS carrinhos_convertidos,
    (COUNT(DISTINCT carrinho_id) - COUNT(DISTINCT CASE WHEN pedido_id IS NOT NULL THEN carrinho_id END)) * 100.0 
        / NULLIF(COUNT(DISTINCT carrinho_id), 0) AS taxa_abandono_pct
FROM Carrinhos
WHERE data_criacao >= DATEADD(DAY, -30, GETDATE());
```

**3. Customer Lifetime Value (CLV)**

$$
CLV = \frac{\text{Ticket Médio} \times \text{Frequência de Compra Anual} \times \text{Tempo de Vida (anos)}}{\text{Taxa de Desconto}}
$$

Versão simplificada (cohort-based):

```python
# Calcular CLV por coorte
df_cohort_ltv = pd.read_sql("""
    WITH PrimeiroPedido AS (
        SELECT 
            cliente_id,
            MIN(data_pedido) AS primeira_compra,
            DATEFROMPARTS(YEAR(MIN(data_pedido)), MONTH(MIN(data_pedido)), 1) AS cohort_mes
        FROM Pedidos
        GROUP BY cliente_id
    ),
    ReceitaPorMes AS (
        SELECT 
            pp.cohort_mes,
            DATEDIFF(MONTH, pp.primeira_compra, p.data_pedido) AS meses_desde_primeira,
            SUM(p.valor_total) AS receita
        FROM Pedidos p
        JOIN PrimeiroPedido pp ON p.cliente_id = pp.cliente_id
        GROUP BY pp.cohort_mes, DATEDIFF(MONTH, pp.primeira_compra, p.data_pedido)
    )
    SELECT 
        cohort_mes,
        meses_desde_primeira,
        receita,
        COUNT(DISTINCT cliente_id) AS clientes
    FROM ReceitaPorMes
    GROUP BY cohort_mes, meses_desde_primeira
""", connection)

# Calcular CLV cumulativo
df_cohort_ltv['clv_cumulativo'] = df_cohort_ltv.groupby('cohort_mes')['receita'].cumsum()

# Visualizar
cohort_clv_pivot = df_cohort_ltv.pivot_table(
    index='cohort_mes',
    columns='meses_desde_primeira',
    values='clv_cumulativo'
)

plt.figure(figsize=(14, 8))
for cohort in cohort_clv_pivot.index[-6:]:  # Últimas 6 coortes
    plt.plot(cohort_clv_pivot.columns, cohort_clv_pivot.loc[cohort], marker='o', label=cohort)

plt.title('CLV Cumulativo por Coorte')
plt.xlabel('Meses desde Primeira Compra')
plt.ylabel('CLV Cumulativo (R$)')
plt.legend(title='Coorte')
plt.grid(True, alpha=0.3)
plt.show()
```

#### Fintech/Banking

**1. Net Interest Margin (NIM)**

$$
NIM = \frac{\text{Receita de Juros} - \text{Custo de Funding}}{\text{Ativos Produtivos}} \times 100\%
$$

```sql
SELECT 
    DATEFROMPARTS(YEAR(data_transacao), MONTH(data_transacao), 1) AS mes,
    SUM(CASE WHEN tipo_transacao = 'receita_juros' THEN valor ELSE 0 END) AS receita_juros,
    SUM(CASE WHEN tipo_transacao = 'custo_funding' THEN valor ELSE 0 END) AS custo_funding,
    AVG(saldo_ativos_produtivos) AS media_ativos,
    (SUM(CASE WHEN tipo_transacao = 'receita_juros' THEN valor ELSE 0 END) 
     - SUM(CASE WHEN tipo_transacao = 'custo_funding' THEN valor ELSE 0 END))
     * 100.0 / NULLIF(AVG(saldo_ativos_produtivos), 0) AS nim_pct
FROM TransacoesFinanceiras
WHERE data_transacao >= DATEADD(YEAR, -1, GETDATE())
GROUP BY DATEFROMPARTS(YEAR(data_transacao), MONTH(data_transacao), 1)
ORDER BY mes;
```

**2. Default Rate (Taxa de Inadimplência)**

$$
\text{Default Rate} = \frac{\text{Empréstimos em Atraso > 90 dias}}{\text{Total de Empréstimos Ativos}} \times 100\%
$$

```sql
WITH StatusEmprestimos AS (
    SELECT 
        emprestimo_id,
        valor_principal,
        data_vencimento,
        CASE 
            WHEN DATEDIFF(DAY, data_vencimento, GETDATE()) > 90 THEN 'Default'
            WHEN DATEDIFF(DAY, data_vencimento, GETDATE()) BETWEEN 30 AND 90 THEN 'Atraso'
            ELSE 'Em Dia'
        END AS status
    FROM Emprestimos
    WHERE status_emprestimo = 'Ativo'
)
SELECT 
    status,
    COUNT(*) AS quantidade,
    SUM(valor_principal) AS valor_total,
    SUM(valor_principal) * 100.0 / SUM(SUM(valor_principal)) OVER () AS percentual
FROM StatusEmprestimos
GROUP BY status;
```

#### SaaS

**1. MRR (Monthly Recurring Revenue)**

$$
MRR = \sum_{i=1}^{n} \text{Valor da Assinatura}_i
$$

**Breakdown de MRR:**
- **Novo MRR**: Novos clientes
- **Expansion MRR**: Upgrades
- **Contraction MRR**: Downgrades
- **Churn MRR**: Cancelamentos

$$
\text{Net New MRR} = \text{Novo} + \text{Expansion} - \text{Contraction} - \text{Churn}
$$

```sql
WITH MRR_Breakdown AS (
    SELECT 
        DATEFROMPARTS(YEAR(data_evento), MONTH(data_evento), 1) AS mes,
        SUM(CASE WHEN tipo_evento = 'nova_assinatura' THEN valor_mensal ELSE 0 END) AS novo_mrr,
        SUM(CASE WHEN tipo_evento = 'upgrade' THEN valor_diferenca ELSE 0 END) AS expansion_mrr,
        SUM(CASE WHEN tipo_evento = 'downgrade' THEN ABS(valor_diferenca) ELSE 0 END) AS contraction_mrr,
        SUM(CASE WHEN tipo_evento = 'cancelamento' THEN valor_mensal ELSE 0 END) AS churn_mrr
    FROM EventosAssinatura
    WHERE data_evento >= DATEADD(YEAR, -1, GETDATE())
    GROUP BY DATEFROMPARTS(YEAR(data_evento), MONTH(data_evento), 1)
)
SELECT 
    mes,
    novo_mrr,
    expansion_mrr,
    contraction_mrr,
    churn_mrr,
    (novo_mrr + expansion_mrr - contraction_mrr - churn_mrr) AS net_new_mrr,
    SUM(novo_mrr + expansion_mrr - contraction_mrr - churn_mrr) OVER (ORDER BY mes) AS mrr_cumulativo
FROM MRR_Breakdown
ORDER BY mes;
```

**2. Churn Rate**

$$
\text{Customer Churn Rate} = \frac{\text{Clientes Perdidos no Mês}}{\text{Clientes no Início do Mês}} \times 100\%
$$

$$
\text{Revenue Churn Rate} = \frac{\text{MRR Perdido no Mês}}{\text{MRR no Início do Mês}} \times 100\%
$$

```python
# Calcular churn rate com análise de sobrevivência
from lifelines import KaplanMeierFitter

df_churn = pd.read_sql("""
    SELECT 
        cliente_id,
        DATEDIFF(MONTH, data_primeira_assinatura, COALESCE(data_cancelamento, GETDATE())) AS tenure_meses,
        CASE WHEN data_cancelamento IS NOT NULL THEN 1 ELSE 0 END AS churned
    FROM Assinaturas
""", connection)

# Kaplan-Meier
kmf = KaplanMeierFitter()
kmf.fit(df_churn['tenure_meses'], df_churn['churned'])

# Plotar curva de sobrevivência
kmf.plot_survival_function()
plt.title('Curva de Sobrevivência - Retenção de Clientes')
plt.xlabel('Meses desde Assinatura')
plt.ylabel('Probabilidade de Retenção')
plt.grid(True, alpha=0.3)
plt.show()

# Probabilidade de retenção em 12 meses
prob_retencao_12m = kmf.survival_function_at_times(12).values[0]
print(f"Probabilidade de retenção em 12 meses: {prob_retencao_12m:.2%}")
```

**3. CAC (Customer Acquisition Cost)**

$$
CAC = \frac{\text{Custo Total de Marketing + Vendas}}{\text{Número de Clientes Adquiridos}}
$$

**Payback Period:**

$$
\text{Payback} = \frac{CAC}{\text{MRR por Cliente}}
$$

```sql
WITH CustoAquisicao AS (
    SELECT 
        DATEFROMPARTS(YEAR(data_custo), MONTH(data_custo), 1) AS mes,
        SUM(valor_custo) AS custo_total
    FROM CustosMarketing
    WHERE data_custo >= DATEADD(YEAR, -1, GETDATE())
    GROUP BY DATEFROMPARTS(YEAR(data_custo), MONTH(data_custo), 1)
),
ClientesAdquiridos AS (
    SELECT 
        DATEFROMPARTS(YEAR(data_primeira_assinatura), MONTH(data_primeira_assinatura), 1) AS mes,
        COUNT(DISTINCT cliente_id) AS novos_clientes,
        AVG(valor_mrr) AS mrr_medio
    FROM Assinaturas
    WHERE data_primeira_assinatura >= DATEADD(YEAR, -1, GETDATE())
    GROUP BY DATEFROMPARTS(YEAR(data_primeira_assinatura), MONTH(data_primeira_assinatura), 1)
)
SELECT 
    ca.mes,
    ca.custo_total,
    cl.novos_clientes,
    ca.custo_total / NULLIF(cl.novos_clientes, 0) AS cac,
    cl.mrr_medio,
    (ca.custo_total / NULLIF(cl.novos_clientes, 0)) / NULLIF(cl.mrr_medio, 0) AS payback_meses
FROM CustoAquisicao ca
JOIN ClientesAdquiridos cl ON ca.mes = cl.mes
ORDER BY ca.mes;
```

**Regra de Ouro SaaS:**

$$
\frac{CLV}{CAC} > 3
$$

Se CLV/CAC < 3, você está gastando muito para adquirir clientes!

### 7.3 KPI Dashboards: A Pirâmide de Métricas

**Estrutura hierárquica:**

```
                    [North Star Metric]
                           |
        +------------------+------------------+
        |                  |                  |
    [KPI 1]            [KPI 2]            [KPI 3]
        |                  |                  |
   +----+----+        +----+----+        +----+----+
   |         |        |         |        |         |
[Métrica] [Métrica] [Métrica] [Métrica] [Métrica] [Métrica]
```

**Exemplo E-commerce:**
- **North Star**: Revenue
  - **KPI 1**: Visitantes × Conversão × Ticket Médio = Revenue
    - Visitantes: Tráfego orgânico, pago, direto
    - Conversão: Taxa por funil, abandono de carrinho
    - Ticket: Produtos por pedido, upsell rate

---

## 8. Dashboards: Storytelling com Dados {#dashboards}

### 8.1 Princípios de Design

#### Lei de Fitts

$$
T = a + b \log_2\left(\frac{D}{W} + 1\right)
$$

onde:
- $T$ = tempo para alcançar alvo
- $D$ = distância ao alvo
- $W$ = largura do alvo

**Aplicação prática**: KPIs críticos devem ser **grandes** e **próximos** do olhar inicial do usuário.

#### Gestalt Principles

1. **Proximidade**: Itens relacionados devem estar próximos
2. **Similaridade**: Itens similares devem ter visual parecido
3. **Figura-Fundo**: Destaque o importante, desfoque o secundário

**❌ Dashboard Ruim:**
```
+----------------------------------+
| KPI1  KPI2  KPI3  KPI4  KPI5    |
| [graf1] [graf2] [graf3] [graf4] |
| [tabela1] [tabela2] [tabela3]   |
+----------------------------------+
```

**✅ Dashboard Bom:**
```
+----------------------------------+
|        [NORTH STAR METRIC]       |
|        Grande, destacado         |
+----------------------------------+
| [KPI 1]  | [KPI 2]  | [KPI 3]   |
| +trend   | +trend   | +trend    |
+--------+-------------------------+
|        |   [Gráfico Principal]  |
| Filtros|   Drilldown interativo |
|        |                        |
+--------+-------------------------+
```

### 8.2 Power BI: Implementação Prática

#### DAX: Medidas Avançadas

**1. YoY Growth (Year-over-Year)**

```dax
YoY_Growth = 
VAR CurrentPeriodRevenue = SUM(Pedidos[valor_total])
VAR PreviousYearRevenue = 
    CALCULATE(
        SUM(Pedidos[valor_total]),
        SAMEPERIODLASTYEAR('Calendario'[Data])
    )
RETURN
DIVIDE(
    CurrentPeriodRevenue - PreviousYearRevenue,
    PreviousYearRevenue,
    0
)
```

**2. Running Total (Acumulado)**

```dax
Running_Total_Revenue = 
CALCULATE(
    SUM(Pedidos[valor_total]),
    FILTER(
        ALL('Calendario'[Data]),
        'Calendario'[Data] <= MAX('Calendario'[Data])
    )
)
```

**3. Moving Average (Média Móvel)**

```dax
MA_7_Days = 
CALCULATE(
    AVERAGE(Pedidos[valor_total]),
    DATESINPERIOD(
        'Calendario'[Data],
        LASTDATE('Calendario'[Data]),
        -7,
        DAY
    )
)
```

**4. Pareto Analysis (80/20)**

```dax
// Primeiro, criar ranking de produtos por receita
Produto_Rank = 
RANKX(
    ALL(Produtos[produto_id]),
    [Total_Receita],
    ,
    DESC,
    DENSE
)

// Depois, calcular contribuição acumulada
Pareto_Acumulado = 
VAR CurrentRank = [Produto_Rank]
RETURN
CALCULATE(
    [Total_Receita],
    FILTER(
        ALL(Produtos[produto_id]),
        [Produto_Rank] <= CurrentRank
    )
) / 
CALCULATE([Total_Receita], ALL(Produtos))

// Identificar se está nos top 20%
Is_Top_20_Percent = 
IF([Pareto_Acumulado] <= 0.8, "Top 20%", "Resto")
```

#### Interatividade: Drillthrough e Bookmarks

**Drillthrough Page para Detalhes de Cliente:**

```
Página Principal: [Card com métrica agregada]
   ↓ Click direito → "Drillthrough para Cliente"
Página Cliente: [RFM Score, Histórico de Compras, Produtos Favoritos]
```

**Configuração:**
1. Criar página "Cliente Detail"
2. Adicionar campo `cliente_id` à área de Drillthrough filters
3. Configurar ação de voltar

**Bookmarks para Alternar Views:**

```dax
// Medida que muda baseada em seletor
Selected_Metric = 
SWITCH(
    SELECTEDVALUE(MetricSelector[Metric]),
    "Revenue", [Total_Revenue],
    "Orders", [Total_Orders],
    "AOV", [Average_Order_Value],
    [Total_Revenue]  // Default
)
```

### 8.3 Storytelling: Do Insight à Ação

#### Estrutura SCQA (Situation, Complication, Question, Answer)

**Exemplo Fintech:**

**S (Situação)**: Nossa taxa de aprovação de crédito está em 65%
**C (Complicação)**: Concorrentes estão em 75%, perdemos market share
**Q (Questão)**: Como aumentar aprovação sem aumentar risco?
**A (Resposta)**: 
- **Insight 1**: 30% das rejeições são por "documentação incompleta" (não risco de crédito)
- **Insight 2**: Clientes com score > 600 e histórico bancário > 6 meses têm default rate < 2%
- **Ação**: Implementar auto-aprovação para esse segmento + simplificar coleta de docs

**Dashboard que conta essa história:**

```
+--------------------------------------------------+
|  Taxa de Aprovação: 65% ↓ (vs 75% mercado)      |
+--------------------------------------------------+
|                                                  |
|  [Gráfico Pizza]                                 |
|  Motivos de Rejeição:                            |
|  • Documentação: 30% ⚠️                          |
|  • Score baixo: 40%                              |
|  • Renda insuficiente: 20%                       |
|  • Outros: 10%                                   |
|                                                  |
+--------------------------------------------------+
|  [Scatter Plot]                                  |
|  Score vs Default Rate (bolhas = volume)         |
|  → Destaque: Score > 600 + hist > 6m = safe zone|
+--------------------------------------------------+
|  RECOMENDAÇÃO:                                   |
|  ✅ Auto-aprovar 12.000 clientes (score > 600)  |
|  💰 Ganho estimado: +R$ 8MM MRR                 |
|  ⚠️ Risco adicional: +0.5% default rate         |
+--------------------------------------------------+
```

---

## 9. Casos Práticos Integrados {#casos-praticos}

### 9.1 Projeto End-to-End: E-commerce Performance Dashboard

**Objetivo**: Construir dashboard executivo que responda:
1. Como está a saúde do negócio hoje?
2. Onde estão as oportunidades de crescimento?
3. Quais ações tomar nos próximos 30 dias?

#### Fase 1: ETL

**Script T-SQL (Carga Incremental):**

```sql
-- Criar tabela de controle de ETL
CREATE TABLE ETL_Control (
    table_name VARCHAR(100),
    last_update_date DATETIME2,
    rows_processed INT
);

-- Procedure de carga incremental
CREATE PROCEDURE sp_Load_Fact_Pedidos
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @LastUpdate DATETIME2;
    SELECT @LastUpdate = last_update_date 
    FROM ETL_Control 
    WHERE table_name = 'Fact_Pedidos';
    
    IF @LastUpdate IS NULL
        SET @LastUpdate = '1900-01-01';
    
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- Insert novos pedidos
        INSERT INTO DW.dbo.Fact_Pedidos (
            pedido_id,
            cliente_key,
            produto_key,
            data_key,
            quantidade,
            valor_unitario,
            valor_total,
            desconto
        )
        SELECT 
            p.pedido_id,
            dc.cliente_key,
            dp.produto_key,
            dd.data_key,
            ip.quantidade,
            ip.preco_unitario,
            ip.quantidade * ip.preco_unitario AS valor_total,
            ip.desconto_percentual
        FROM Producao.dbo.Pedidos p
        JOIN Producao.dbo.ItensPedido ip ON p.pedido_id = ip.pedido_id
        JOIN DW.dbo.Dim_Cliente dc ON p.cliente_id = dc.cliente_id
        JOIN DW.dbo.Dim_Produto dp ON ip.produto_id = dp.produto_id
        JOIN DW.dbo.Dim_Data dd ON CAST(p.data_pedido AS DATE) = dd.data_completa
        WHERE p.data_atualizacao > @LastUpdate;
        
        -- Update controle
        UPDATE ETL_Control
        SET 
            last_update_date = GETDATE(),
            rows_processed = @@ROWCOUNT
        WHERE table_name = 'Fact_Pedidos';
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        -- Log de erro
        INSERT INTO ETL_ErrorLog (error_message, error_date)
        VALUES (ERROR_MESSAGE(), GETDATE());
    END CATCH;
END;
```

#### Fase 2: EDA em Python

```python
import pyodbc
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats

# Conexão
conn = pyodbc.connect('DRIVER={SQL Server};SERVER=your_server;DATABASE=DW;Trusted_Connection=yes;')

# Extrair dados agregados
query = """
SELECT 
    dd.ano,
    dd.mes,
    dd.data_completa,
    SUM(fp.valor_total) AS receita,
    COUNT(DISTINCT fp.pedido_id) AS pedidos,
    COUNT(DISTINCT fp.cliente_key) AS clientes_unicos,
    AVG(fp.valor_total) AS ticket_medio
FROM Fact_Pedidos fp
JOIN Dim_Data dd ON fp.data_key = dd.data_key
WHERE dd.data_completa >= DATEADD(YEAR, -2, GETDATE())
GROUP BY dd.ano, dd.mes, dd.data_completa
ORDER BY dd.data_completa
"""

df = pd.read_sql(query, conn)
df['data_completa'] = pd.to_datetime(df['data_completa'])

# EDA: Tendência de receita
plt.figure(figsize=(14, 6))
plt.plot(df['data_completa'], df['receita'], marker='o')
plt.title('Tendência de Receita - Últimos 24 Meses')
plt.xlabel('Data')
plt.ylabel('Receita (R$)')
plt.xticks(rotation=45)
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()

# Decomposição de série temporal
from statsmodels.tsa.seasonal import seasonal_decompose

df_ts = df.set_index('data_completa')
decomposition = seasonal_decompose(df_ts['receita'], model='additive', period=12)

fig, axes = plt.subplots(4, 1, figsize=(14, 10))
decomposition.observed.plot(ax=axes[0], title='Observado')
decomposition.trend.plot(ax=axes[1], title='Tendência')
decomposition.seasonal.plot(ax=axes[2], title='Sazonalidade')
decomposition.resid.plot(ax=axes[3], title='Resíduo')
plt.tight_layout()
plt.show()

# Teste de estacionariedade (Augmented Dickey-Fuller)
from statsmodels.tsa.stattools import adfuller

result = adfuller(df['receita'].dropna())
print(f'ADF Statistic: {result[0]}')
print(f'p-value: {result[1]}')

if result[1] < 0.05:
    print('✅ Série é estacionária')
else:
    print('❌ Série não é estacionária - precisa diferenciar')
    
    # Diferenciar
    df['receita_diff'] = df['receita'].diff()
    result_diff = adfuller(df['receita_diff'].dropna())
    print(f'\nApós diferenciação:')
    print(f'ADF Statistic: {result_diff[0]}')
    print(f'p-value: {result_diff[1]}')
```

#### Fase 3: Segmentação RFM + Predição de Churn

```python
# RFM
from datetime import datetime

data_referencia = datetime.now()

query_rfm = """
SELECT 
    cliente_key,
    DATEDIFF(DAY, MAX(dd.data_completa), GETDATE()) AS recency,
    COUNT(DISTINCT pedido_id) AS frequency,
    SUM(valor_total) AS monetary
FROM Fact_Pedidos fp
JOIN Dim_Data dd ON fp.data_key = dd.data_key
WHERE dd.data_completa >= DATEADD(YEAR, -1, GETDATE())
GROUP BY cliente_key
"""

df_rfm = pd.read_sql(query_rfm, conn)

# Criar scores RFM
df_rfm['R_Score'] = pd.qcut(df_rfm['recency'], q=5, labels=[5,4,3,2,1])
df_rfm['F_Score'] = pd.qcut(df_rfm['frequency'].rank(method='first'), q=5, labels=[1,2,3,4,5])
df_rfm['M_Score'] = pd.qcut(df_rfm['monetary'], q=5, labels=[1,2,3,4,5])

df_rfm['RFM_Score'] = df_rfm['R_Score'].astype(str) + df_rfm['F_Score'].astype(str) + df_rfm['M_Score'].astype(str)

# Segmentação
def rfm_segment(row):
    r, f, m = int(row['R_Score']), int(row['F_Score']), int(row['M_Score'])
    
    if r >= 4 and f >= 4 and m >= 4:
        return 'Champions'
    elif r >= 3 and f >= 3 and m >= 3:
        return 'Loyal Customers'
    elif r >= 4 and f <= 2:
        return 'New Customers'
    elif r <= 2 and f >= 3:
        return 'At Risk'
    elif r <= 2 and f <= 2:
        return 'Lost'
    else:
        return 'Others'

df_rfm['Segment'] = df_rfm.apply(rfm_segment, axis=1)

# Modelo de Churn (Logistic Regression)
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report, confusion_matrix, roc_auc_score, roc_curve

# Preparar dados
df_rfm['Churned'] = (df_rfm['recency'] > 180).astype(int)  # 6 meses sem comprar = churn

X = df_rfm[['recency', 'frequency', 'monetary']]
y = df_rfm['Churned']

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=42)

# Treinar modelo
model = LogisticRegression(random_state=42, max_iter=1000)
model.fit(X_train, y_train)

# Predições
y_pred = model.predict(X_test)
y_pred_proba = model.predict_proba(X_test)[:, 1]

# Avaliar
print("Confusion Matrix:")
print(confusion_matrix(y_test, y_pred))
print("\nClassification Report:")
print(classification_report(y_test, y_pred))
print(f"\nROC AUC Score: {roc_auc_score(y_test, y_pred_proba):.4f}")

# Curva ROC
fpr, tpr, thresholds = roc_curve(y_test, y_pred_proba)

plt.figure(figsize=(10, 6))
plt.plot(fpr, tpr, label=f'ROC Curve (AUC = {roc_auc_score(y_test, y_pred_proba):.2f})')
plt.plot([0, 1], [0, 1], 'k--', label='Random Classifier')
plt.xlabel('False Positive Rate')
plt.ylabel('True Positive Rate')
plt.title('ROC Curve - Churn Prediction')
plt.legend()
plt.grid(True, alpha=0.3)
plt.show()

# Aplicar modelo em toda base
df_rfm['Churn_Probability'] = model.predict_proba(df_rfm[['recency', 'frequency', 'monetary']])[:, 1]
df_rfm['Churn_Risk'] = pd.cut(df_rfm['Churn_Probability'], bins=[0, 0.3, 0.7, 1], labels=['Low', 'Medium', 'High'])
```

#### Fase 4: Dashboard em Power BI

**Estrutura:**

**Página 1: Visão Executiva**
- Card: Receita MTD vs Target
- Card: Pedidos MTD vs Target  
- Card: Ticket Médio vs Baseline
- Gráfico de Linha: Receita Diária (30 dias) com média móvel
- Gráfico de Barras: Top 5 Produtos por Receita
- Waterfall Chart: Breakdown de receita (novos clientes, recorrentes, upsell)

**Página 2: Análise de Clientes**
- Heatmap RFM (Recency vs Frequency)
- Treemap: Segmentos RFM por tamanho
- Scatter Plot: CLV vs CAC por coorte
- Tabela: Top 100 clientes por risco de churn (ordenado por Churn_Probability DESC)

**Página 3: Análise de Produto**
- Gráfico de Pareto: Produtos (80/20)
- Matriz de correlação: Cross-sell (produtos comprados juntos)
- Funnel: Jornada do produto (visualizações → carrinho → compra)

**DAX Measures:**

```dax
// Target Revenue (baseado em forecast)
Target_Revenue = 
VAR CurrentMonth = MONTH(TODAY())
VAR CurrentYear = YEAR(TODAY())
RETURN
CALCULATE(
    SUM(Forecast[target_revenue]),
    FILTER(
        Forecast,
        Forecast[mes] = CurrentMonth && Forecast[ano] = CurrentYear
    )
)

// Revenue vs Target %
Revenue_vs_Target_Pct = 
DIVIDE([Total_Revenue], [Target_Revenue], 0) - 1

// Conditional formatting (vermelho se < 90%, verde se > 100%)
Revenue_Color = 
SWITCH(
    TRUE(),
    [Revenue_vs_Target_Pct] < -0.1, "Red",
    [Revenue_vs_Target_Pct] > 0, "Green",
    "Orange"
)
```

---

## 10. Armadilhas e Boas Práticas {#armadilhas}

### 10.1 Armadilhas Estatísticas

#### 1. Paradoxo de Simpson

**Exemplo**: Taxa de conversão por canal

```
Canal A: 400/1000 conversões (40%)
Canal B: 150/500 conversões (30%)

Conclusão: Canal A é melhor?
```

**Mas, segmentando por dispositivo:**

```
           Mobile              Desktop
Canal A: 100/500 (20%)    300/500 (60%)
Canal B: 50/100 (50%)     100/400 (25%)
```

Canal B é **melhor em ambos** os segmentos! O paradoxo ocorre porque Canal A tem mais tráfego desktop (alto conv.) enquanto Canal B tem mais mobile (baixo conv.).

**Lição**: **Sempre segmentar** antes de tirar conclusões.

#### 2. Selection Bias (Viés de Seleção)

**Exemplo Fintech**: "Clientes que pagam empréstimos em dia têm score médio de 750"

**Problema**: Você só está olhando quem **foi aprovado**! Há milhares com score 750+ que foram **rejeitados** por outros critérios.

**Solução**: Incluir **toda a população** no denominador:
- Score 750+, aprovados, pagaram em dia
- Score 750+, aprovados, deram default
- Score 750+, **rejeitados**

#### 3. Survivorship Bias

**Exemplo SaaS**: "Nossos clientes enterprise têm churn de apenas 5%!"

**Problema**: Só está medindo quem **sobreviveu** até virar enterprise. Os que cancelaram antes não entraram na conta.

**Solução**: Cohort analysis desde a aquisição.

### 10.2 Armadilhas de Engenharia de Dados

#### 1. Join sem validação

```sql
-- ❌ PERIGOSO
SELECT 
    p.pedido_id,
    c.nome
FROM Pedidos p
JOIN Clientes c ON p.cliente_id = c.cliente_id;
-- E se houver pedidos órfãos (sem cliente)? Eles desaparecem silenciosamente!

-- ✅ SEGURO
SELECT 
    p.pedido_id,
    COALESCE(c.nome, 'CLIENTE NÃO ENCONTRADO') AS nome,
    CASE WHEN c.cliente_id IS NULL THEN 1 ELSE 0 END AS is_orfao
FROM Pedidos p
LEFT JOIN Clientes c ON p.cliente_id = c.cliente_id;

-- Alertar se houver órfãos
IF EXISTS (SELECT 1 FROM Pedidos p WHERE NOT EXISTS (SELECT 1 FROM Clientes c WHERE c.cliente_id = p.cliente_id))
BEGIN
    RAISERROR('⚠️ ALERTA: Pedidos órfãos detectados!', 16, 1);
END
```

#### 2. Timezone Hell

**Problema**: Servidor em UTC, usuários em GMT-3, logs em GMT-5...

```sql
-- ❌ ERRADO
SELECT data_pedido FROM Pedidos;  -- Que timezone é essa?

-- ✅ CORRETO
-- Sempre converter para timezone padrão
SELECT 
    data_pedido_utc,
    data_pedido_utc AT TIME ZONE 'UTC' AT TIME ZONE 'E. South America Standard Time' AS data_pedido_brt
FROM Pedidos;
```

#### 3. Aggregation antes de Join

```sql
-- ❌ LENTO
SELECT 
    c.cliente_id,
    c.nome,
    SUM(p.valor_total) AS total_gasto
FROM Clientes c
LEFT JOIN Pedidos p ON c.cliente_id = p.cliente_id
GROUP BY c.cliente_id, c.nome;
-- Problema: Faz join de TODAS as linhas primeiro

-- ✅ RÁPIDO
WITH Agregado AS (
    SELECT 
        cliente_id,
        SUM(valor_total) AS total_gasto
    FROM Pedidos
    GROUP BY cliente_id
)
SELECT 
    c.cliente_id,
    c.nome,
    COALESCE(a.total_gasto, 0) AS total_gasto
FROM Clientes c
LEFT JOIN Agregado a ON c.cliente_id = a.cliente_id;
-- Agrega primeiro, depois join apenas linhas agregadas
```

### 10.3 Boas Práticas

#### 1. Documentação como Código

```sql
-- Criar dicionário de dados
CREATE TABLE Data_Dictionary (
    table_name VARCHAR(100),
    column_name VARCHAR(100),
    data_type VARCHAR(50),
    description NVARCHAR(500),
    business_owner VARCHAR(100),
    last_updated DATE
);

-- Exemplo de registro
INSERT INTO Data_Dictionary VALUES 
('Fact_Pedidos', 'valor_total', 'DECIMAL(10,2)', 
 'Valor total do pedido INCLUINDO impostos e frete', 
 'time-produto@empresa.com', 
 '2024-01-15');
```

#### 2. Testes Automatizados

```sql
-- Criar suite de testes de qualidade de dados
CREATE TABLE DQ_Tests (
    test_id INT IDENTITY(1,1),
    test_name VARCHAR(200),
    test_query NVARCHAR(MAX),
    expected_result VARCHAR(50),
    severity VARCHAR(20)  -- Critical, Warning, Info
);

-- Exemplo: Teste de completude
INSERT INTO DQ_Tests (test_name, test_query, expected_result, severity) VALUES 
(
    'Pedidos sem cliente',
    'SELECT COUNT(*) FROM Pedidos p WHERE NOT EXISTS (SELECT 1 FROM Clientes c WHERE c.cliente_id = p.cliente_id)',
    '0',
    'Critical'
);

-- Procedure para rodar testes
CREATE PROCEDURE sp_Run_DQ_Tests
AS
BEGIN
    DECLARE @test_id INT, @test_query NVARCHAR(MAX), @expected VARCHAR(50), @actual VARCHAR(50);
    
    DECLARE test_cursor CURSOR FOR SELECT test_id, test_query, expected_result FROM DQ_Tests;
    OPEN test_cursor;
    
    FETCH NEXT FROM test_cursor INTO @test_id, @test_query, @expected;
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Executar teste e comparar resultado
        EXEC sp_executesql @test_query, N'@result VARCHAR(50) OUTPUT', @actual OUTPUT;
        
        IF @actual <> @expected
        BEGIN
            RAISERROR('Test %d FAILED: Expected %s, got %s', 16, 1, @test_id, @expected, @actual);
        END
        
        FETCH NEXT FROM test_cursor INTO @test_id, @test_query, @expected;
    END
    
    CLOSE test_cursor;
    DEALLOCATE test_cursor;
END;
```

#### 3. Versionamento de Modelos

```python
# Salvar modelo com metadata
import joblib
import json
from datetime import datetime

# Treinar modelo
model = LogisticRegression()
model.fit(X_train, y_train)

# Metadata
metadata = {
    'model_type': 'LogisticRegression',
    'features': list(X_train.columns),
    'training_date': datetime.now().isoformat(),
    'train_size': len(X_train),
    'test_size': len(X_test),
    'metrics': {
        'accuracy': accuracy_score(y_test, model.predict(X_test)),
        'roc_auc': roc_auc_score(y_test, model.predict_proba(X_test)[:, 1])
    },
    'hyperparameters': model.get_params()
}

# Salvar
model_version = '1.0.0'
joblib.dump(model, f'models/churn_model_v{model_version}.pkl')

with open(f'models/churn_model_v{model_version}_metadata.json', 'w') as f:
    json.dump(metadata, f, indent=4)

print(f"Model v{model_version} saved with metadata")
```

---

## Conceitos de Cloud (Bonus)

### Arquitetura Cloud-Native para Analytics

**Lambda Architecture:**

```
Streaming Data → Kafka → Spark Streaming → Hot Storage (Redis)
                    ↓
Batch Data → ETL → Data Lake (S3) → Spark Batch → Cold Storage (Snowflake)
                                                          ↓
                                                   Analytics Layer (Power BI)
```

**Benefícios:**
- **Escalabilidade**: Spark processa terabytes distribuído
- **Custo**: S3 custa centavos por GB, Snowflake cobra por uso
- **Latência**: Redis para métricas real-time, Snowflake para histórico

**Equivalentes Microsoft (já que você usa SQL Server):**
- **Azure Data Factory** = Orquestração de ETL (substituiria SQL Agent)
- **Azure Synapse Analytics** = Data Warehouse cloud (substituiria SQL Server on-prem)
- **Azure Blob Storage** = Data Lake
- **Azure Stream Analytics** = Processamento de streaming

**Quando migrar para cloud?**
- Volume > 10TB (storage on-prem fica caro)
- Necessidade de processamento paralelo massivo
- Time global (latência em diferentes regiões)

---

## Conclusão: O Mindset do Analista Sênior

### 1. Questione Tudo
- "Essa métrica está correta?" → Valide com diferentes fontes
- "Esse insight é acionável?" → Se não guia decisão, é vaidade
- "Estou sendo rigoroso?" → Use matemática, não intuição

### 2. Comunique com Claridade
- Gráfico ≠ Insight
- Insight ≠ Ação
- **Exemplo**: "Churn subiu 10%" (dado) → "Clientes de plano básico estão cancelando após 3 meses" (insight) → "Oferecer upgrade com desconto no 2º mês" (ação)

### 3. Equilibre Velocidade e Perfeição
- **80/20**: 20% do esforço entrega 80% do valor
- **Iterate**: MVP de dashboard → feedback → refinamento
- **Automatize**: Se você faz > 2x, escreva um script

### 4. Seja Tradutor
- **Stakeholder diz**: "Precisamos aumentar vendas"
- **Você traduz para**: "Quais alavancas? Novos clientes (CAC), retenção (churn), ticket médio (upsell)?"
- **Você entrega**: Dashboard com breakdown + recomendações priorizadas por ROI

---

**Este guia é apenas o começo. A maestria vem de:**
- **Fazer**: Implemente cada conceito em um projeto real
- **Errar**: Quebre dashboards, faça joins errados, tire conclusões precipitadas
- **Aprender**: Cada erro é uma armadilha que você não cairá de novo
- **Compartilhar**: Ensine o que aprendeu - é a melhor forma de consolidar

**Recursos para Continuar:**
- Livros: "Storytelling with Data" (Cole Nussbaumer), "The Data Warehouse Toolkit" (Ralph Kimball)
- Cursos: Coursera (Statistics, Machine Learning), Mode Analytics SQL School
- Prática: Kaggle datasets, Análise exploratória de bases públicas (IBGE, Banco Central)

**Boa jornada! 🚀**
