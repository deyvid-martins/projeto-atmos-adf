# ⛅ Atmos ADF — Ingestão Multi-Cloud & Orquestração em Nuvem

![Azure Data Factory](https://img.shields.io/badge/Azure_Data_Factory-0089D6?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Azure Key Vault](https://img.shields.io/badge/Azure_Key_Vault-0089D6?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Google BigQuery](https://img.shields.io/badge/Google_BigQuery-4285F4?style=for-the-badge&logo=googlecloud&logoColor=white)
![GitHub CI/CD](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)

> **Objetivo Profissional:** Documentação técnica de arquitetura e ingestão de dados em nuvem voltada para **Engenharia de Dados**, **Análise de Dados** e **Analytics Engineering**.  
> **Tecnologias Utilizadas:** Microsoft Azure (Azure Data Factory, Data Lake Storage Gen2, Key Vault), Google Cloud Platform (BigQuery, Service Accounts), GitHub, REST APIs (Visual Crossing Weather API) e Postman.

---

## 📌 Visão Geral da Arquitetura & Propósito do Projeto

O projeto **Atmos ADF** tem como objetivo construir um ecossistema de ingestão e integração de dados climáticos e meteorológicos em nuvem híbrida (*Multi-Cloud*), combinando:
1. **Dados em Tempo Real / Diários (API REST):** Captura de métricas meteorológicas atuais via *Visual Crossing Weather API*.
2. **Dados Históricos (Google BigQuery):** Carga de dados históricos de estações meteorológicas brasileiras (INMET) oriundos do *Base dos Dados* no GCP.
3. **Data Lake Hub (Azure Data Lake Storage Gen2):** Armazenamento em camada *Landing Zone* nos formatos JSON e Parquet, aplicando particionamento dinâmico.

```
+------------------------------------+       +-----------------------------------+
|  Source 1: Visual Crossing API     |       |  Source 2: GCP BigQuery (INMET)   |
|  (Dados Climáticos em Tempo Real)  |       |  (Dados Históricos de Microdados) |
+------------------------------------+       +-----------------------------------+
                  |                                            |
                  v                                            v
+--------------------------------------------------------------------------------+
|                          AZURE DATA FACTORY (ADF)                              |
|  - Managed Identity & Secret Management via Azure Key Vault                    |
|  - Parameterized Pipelines, Dynamic Querying & Control Flow (ForEach, Range)  |
|  - GitHub CI/CD Integration (Branch-based Workflow)                            |
+--------------------------------------------------------------------------------+
                                       |
                                       v
+--------------------------------------------------------------------------------+
|                AZURE DATA LAKE STORAGE GEN2 (ADLS Gen2)                        |
|  - Container: atmos-landing-dev-001v2                                          |
|  - Format: JSON (API) & Parquet (INMET)                                        |
|  - Hive Partitioning: /landing/visualcrossing/city/year=YYYY/month=MM/day=DD   |
|                        /landing/inmet/microdados/id_estacao/ano=YYYY           |
+--------------------------------------------------------------------------------+
```

---

## 🚩 Guia Etapa por Etapa: Sequência Cronológica e Boas Práticas

---

### **Fase 1: Governança, Segurança e Controle de Versão**

#### **Etapa 1 — Criação da Conta Azure e Resource Group**
- **O que foi feito:** Ativação da conta *Free Tier* na Azure e criação do primeiro Grupo de Recursos (*Resource Group*).
- **Recurso Criado:** `rg-atmos-dev-westus2-001`
- **Padrão de Nomenclatura Aplicado:**  
  `rg-<projeto>-<ambiente>-<regiao>-<instancia>`
- **Por que esta etapa é vital?**  
  Um *Resource Group* atua como uma barreira lógica de organização e governança. Agrupar os recursos por projeto e ambiente (`dev`, `prod`) facilita a gestão de custos, aplicação de políticas de segurança (RBAC) e remoção limpa de recursos ao término do projeto.

---

#### **Etapa 2 — Criação do Azure Data Lake Storage Gen2 (ADLS Gen2)**
- **O que foi feito:** Provisionamento de uma conta de armazenamento com funcionalidade de *Hierarchical Namespace* ativada (Data Lake Gen2) e criação do container *Landing Zone*.
- **Storage Account:** `saatmosdevwestus2001v2` *(sem caracteres especiais, pois a Azure limita a letras minúsculas e números)*
- **Container:** `atmos-landing-dev-001v2`
- **Por que esta etapa é vital?**  
  O ADLS Gen2 combina o baixo custo do Blob Storage com um sistema de arquivos hierárquico otimizado para motores de BigQuery/Data Analytics (como Azure Synapse e Databricks). A camada **Landing** serve para armazenar os dados brutos exatamente como chegam da fonte (*Raw Layer*), garantindo auditabilidade e imutabilidade dos dados.

---

#### **Etapa 3 — Provisionamento do Azure Key Vault (AKV)**
- **O que foi feito:** Criação do cofre de chaves centralizado para gerenciamento seguro de credenciais e segredos.
- **Recurso Criado:** `kv-atmos-dev-westus2-00`
- **Por que esta etapa é vital?**  
  Regra fundamental de Engenharia de Dados: **nunca grave credenciais, senhas ou API Keys diretamente no código do pipeline (*hardcoding*)**. O Azure Key Vault isola as credenciais, permitindo rotação de chaves e acesso auditado via permissões de identidade.

---

#### **Etapa 4 e 5 — Provisionamento do ADF e Integração com GitHub**
- **O que foi feito:** Instalação do Azure Data Factory (`adf-atmos-dev-westus2-001v2`) integrado nativamente a um repositório GitHub, criando a branch de desenvolvimento/publicação `adf-publish`.
- **Por que esta etapa é vital?**  
  A integração com o GitHub implementa práticas modernas de **DataOps** e **CI/CD**. Trabalhar em branches isoladas impede que alterações incompletas afetem o ambiente de execução e garante controle de versão e rastreabilidade sobre a infraestrutura como código (JSONs dos pipelines).

---

### **Fase 2: Conectividade Segura e Integração com a API Weather (Visual Crossing)**

#### **Etapa 6 — Análise da Fonte de Dados REST API (Visual Crossing)**
- **O que foi feito:** Teste preliminar da API via Postman para entender a estrutura da resposta JSON, parâmetros obrigatórios e limite de requisições.
- **URL Base de Teste:**
  ```http
  GET https://weather.visualcrossing.com/VisualCrossingWebServices/rest/services/timeline/Brasilia/2026-08-05?unitGroup=metric&include=days&key=
  ```
- **Por que esta etapa é vital?**  
  Antes de construir pipelines automatizados, o Engenheiro de Dados deve mapear os contratos da API (métodos HTTP, codificação de paginação, limites de *rate-limiting* e schema da resposta) para garantir que as funções dinâmicas do ADF interpretem corretamente os dados.

---

#### **Etapa 7 — Configuração do Linked Service HTTP e ADLS Gen2 com Identity Management**
- **O que foi feito:** Configuração dos *Linked Services* no ADF. Para o ADLS Gen2, concedeu-se a permissão de RBAC **Storage Blob Data Contributor** para a *Managed Identity (MSI)* do Data Factory.
- **Por que esta etapa é vital?**  
  Ao utilizar *Managed Identity*, o ADF se autentica no Storage Account sem a necessidade de chaves de acesso (*Account Keys*) manuais. Isso garante uma comunicação segura e em conformidade com o princípio de privilégio mínimo.

---

#### **Etapa 8 — Criação do Dataset Parametrizado da API (Source)**
- **O que foi feito:** Criação do dataset HTTP em formato JSON com convenção rigorosa de nomes:  
  `ds_json_visualcrossing_src_dev`
- **Parâmetros Definidos:** `city`, `date`, `apiKey`
- **URL Relativa Dinâmica:**
  ```expression
  @concat(dataset().city, '/', dataset().date, '?unitGroup=metric&include=days&key=', dataset().apiKey)
  ```
- **Por que esta etapa é vital?**  
  Datasets reiproveitáveis e parametrizados evitam a duplicação de componentes no ADF. Um único dataset atende a chamadas de qualquer cidade ou data.

---

#### **Etapa 9 — Criação do Dataset Landing Partitioned (Sink)**
- **O que foi feito:** Criação do dataset Azure Blob FS / Gen2 em formato JSON:  
  `ds_json_visualcrossing_landing_dev`
- **Caminho do Direitório Dinâmico (Hive Partitioning):**
  ```expression
  @tolower(concat(
    'landing/visualcrossing/',
    dataset().city,
    '/year=', formatDateTime(dataset().data, 'yyyy'),
    '/month=', formatDateTime(dataset().data, 'MM'),
    '/day=', formatDateTime(dataset().data, 'dd')
  ))
  ```
- **Por que esta etapa é vital?**  
  O padrão de particionamento estilo Hive (`year=YYYY/month=MM/day=DD`) otimiza drasticamente as futuras consultas em motores analíticos (Databricks, Synapse, Athena), pois permite o *Partition Pruning* (leitura apenas das pastas relevantes para a consulta, reduzindo custos e tempo de processamento).

---

#### **Etapa 10 — Construção do Pipeline Baseline de Ingestão (`pl_ingest_visualcrossing_landing_dev`)**
- **O que foi feito:** Desenvolvimento da atividade *Copy Data* (`copy_visualcrossing_to_landing`).
- **Nomenclatura do Arquivo no Sink:**
  ```expression
  @tolower(concat('atmos_', pipeline().parameters.city, '_', formatDateTime(utcNow(), 'yyyyMMddhhmmss'), '.json'))
  ```
- **Por que esta etapa é vital?**  
  Gerar nomes de arquivos contendo *timestamps* e identificadores evita a sobreescrita de dados (*data loss*) e garante o rastreamento da exata data/hora em que a ingestão ocorreu.

---

### **Fase 3: Refatoração para Segurança Avançada e Carga Histórica Dinâmica**

#### **Etapa 11 e 12 — Integração do Azure Key Vault e Atividade Web no ADF**
- **O que foi feito:**
  1. Criação do segredo `visualcrossing-api-key-dev` dentro do Azure Key Vault (`kv-atmos-dev-westus2-00`).
  2. Conexão do ADF ao AKV criando o Linked Service `ls_akv_atmos_dev` via *Managed Identity*.
  3. Adição de uma atividade **Web** no pipeline para resgatar dinamicamente a API Key no momento da execução:
     - **URL:** `https://kv-atmos-dev-westus2-00.vault.azure.net/secrets/visualcrossing-api-key-dev/?api-version=7.4`
     - **Authentication:** System-assigned managed identity
     - **Resource:** `https://vault.azure.net`
  4. Atribuição do valor recuperado diretamente ao parâmetro do Copy Data:  
     `@activity('web_get_apikey').output.value`
- **Por que esta etapa é vital?**  
  Elimina o risco de vazamento de segredos no GitHub ou nos logs do Data Factory. O pipeline lê o segredo diretamente da memória em tempo de execução.

---

#### **Etapa 13 — Ajuste do Pipeline Diário (`Daily Pipeline`)**
- **O que foi feito:** Adição da atividade *Set Variable* (`current_date`) utilizando expressões de data atual do sistema para garantir cargas executadas automaticamente todos os dias sem intervenção manual.

---

#### **Etapa 14 — Desenvolvimento do Pipeline de Carga Histórica (`pl_ingest_visualcrossing_landing_dev_history`)**
- **O que foi feito:** Refatoração do pipeline para ler intervalos arbitrários de datas passadas.
  1. **Parâmetros de Entrada:** `startDate` e `endDate`.
  2. **Geração da Lista de Intervalo de Dias (*Array*):**
     ```expression
     @range(0, add(div(sub(ticks(pipeline().parameters.endDate), ticks(pipeline().parameters.startDate)), 864000000000), 1))
     ```
  3. **Iteração Controlada com ForEach Activity:** Configuração de *Batch Count* = `10` (para evitar *rate limiting* da API).
  4. **Atribuição Dinâmica de Data na Iteração:**
     ```expression
     @formatDateTime(addDays(pipeline().parameters.startDate, item()), 'yyyy-MM-dd')
     ```
- **Por que esta etapa é vital?**  
  Em projetos de Analytics e Engenharia de Dados, a capacidade de realizar *Backfill* (re-processamento de dados históricos) de forma automatizada e parametrizada é uma habilidade fundamental.

---

#### **Etapa 15 — Automação via Triggers e Monitoramento do Pipeline**
- **O que foi feito:** Configuração de *Schedule Triggers* no Data Factory para execução diária agendada e utilização da aba *Monitor* para inspeção de métricas de sucesso, duração e diagnóstico de falhas.

---

### **Fase 4: Integração Híbrida GCP BigQuery & Armazenamento em Parquet**

#### **Etapa 16 e 17 — Configuração do Google Cloud Platform (GCP) e Mapeamento dos Dados Públicos do INMET**
- **O que foi feito:**
  1. Criação do projeto `atmos-505916` na GCP.
  2. Consulta do dataset público `basedosdados.br_inmet_bdmep.microdados` via BigQuery Studio para validação dos dados da estação meteorológica `A014` (Goiás).
- **Consulta de Validação SQL:**
  ```sql
  SELECT 
    ano,
    COUNT(*) AS total_registros
  FROM `basedosdados.br_inmet_bdmep.microdados`
  WHERE id_estacao = 'A014'
  GROUP BY ano
  ORDER BY ano;
  ```
- **Por que esta etapa é vital?**  
  A validação prévia dos volume de dados e cobertura temporal (2007–2026) permite dimensionar o processamento e definir a melhor estratégia de ingestão para a nuvem Azure.

---

#### **Etapa 18 — Conectividade Multi-Cloud: Linked Service BigQuery e Governança GCP (Org Policy)**
- **O que foi feito:**
  1. Criação da *Service Account* `adf-bigquery-reader` no GCP com as permissões restritas:
     - `BigQuery Data Viewer`
     - `BigQuery Job User`
     - `BigQuery Read Session User`
  2. **Resolução de Bloqueio de Segurança (Org Policy):** Alteração da regra `iam.disableServiceAccountKeyCreation` no GCP para viabilizar o download seguro da chave de serviço em formato JSON.
  3. **Armazenamento Seguro da Chave:** Registro da chave JSON inteira no Azure Key Vault (`adf-bigquery-reader`).
  4. Criação do Linked Service `ls_bigquery_inmet_dev` no ADF consumindo o segredo do AKV.
- **Por que esta etapa é vital?**  
  Demonstra maturidade técnica na resolução de problemas complexos de governança corporativa entre ecossistemas concorrentes (Azure vs GCP), aplicando segurança de privilégio mínimo em ambos os lados.

---

#### **Etapa 19 e 20 — Pipeline de Ingestão de Dados do INMET em Formato Parquet**
- **O que foi feito:**
  1. **Dataset Source (BigQuery):** `adf_bigquery_inmet_src_dev` com parâmetros `id_estacao` e `ano`.
  2. **Dataset Sink (ADLS Gen2):** `ds_parquet_inmet_landing_dev` gravando dados em **Apache Parquet**.
  3. **Pipeline `pl_ingest_inmet_landing_dev`:**
     - Geração dinâmica do intervalo de anos:
       ```expression
       @range(pipeline().parameters.startYear, add(sub(pipeline().parameters.endYear, pipeline().parameters.startYear), 1))
       ```
     - Consulta SQL Dinâmica (GoogleSQL no Source):
       ```expression
       @concat('SELECT * FROM `basedosdados.br_inmet_bdmep.microdados` WHERE id_estacao = ''', pipeline().parameters.id_estacao, ''' AND ano = ', string(item()))
       ```
     - Gravação no Data Lake com particionamento por ano em formato colunar Parquet:
       ```expression
       @tolower(concat('atmos_', pipeline().parameters.id_estacao, '_', string(item()), formatDateTime(utcNow(), 'yyyyMMddhhmmss'), '.parquet'))
       ```
- **Por que esta etapa é vital?**  
  O formato **Parquet** é o padrão-ouro na Engenharia de Dados para dados estruturados. Por ser um formato de armazenamento colunar com compressão nativa (Snappy/Gzip), ele reduz o custo de armazenamento no Data Lake e acelera o tempo de execução de queries em ferramentas como Azure Databricks e PySpark.

---

## 🏗️ Resumo dos Padrões de Nomenclatura (Naming Conventions)

| Categoria | Recurso | Padrão Utilizado | Exemplo |
| :--- | :--- | :--- | :--- |
| **Infraestrutura** | Resource Group | `rg-<projeto>-<ambiente>-<regiao>-<instancia>` | `rg-atmos-dev-westus2-001` |
| **Armazenamento** | Storage Account | `sa<projeto><ambiente><regiao><instancia>` | `saatmosdevwestus2001v2` |
| **Segurança** | Key Vault | `kv-<projeto>-<ambiente>-<regiao>-<instancia>` | `kv-atmos-dev-westus2-00` |
| **Orquestração** | Data Factory | `adf-<projeto>-<ambiente>-<regiao>-<instancia>` | `adf-atmos-dev-westus2-001v2` |
| **Conectividade** | Linked Service | `ls_<fonte/servico>_<detalhe>_<ambiente>` | `ls_akv_atmos_dev` / `ls_bigquery_inmet_dev` |
| **Datasets** | Datasets ADF | `ds_<formato>_<fonte>_<papel>_<ambiente>` | `ds_json_visualcrossing_src_dev` / `ds_parquet_inmet_landing_dev` |
| **Pipelines** | Pipelines ADF | `pl_<acao>_<fonte>_<camada>_<ambiente>` | `pl_ingest_visualcrossing_landing_dev` |

---

## 🎯 Próximos Passos do Projeto (Roadmap Técnico)

1. **Camada Bronze / Silver no Azure Databricks:**
   - Consumo dos arquivos brutos (JSON e Parquet) gravados na *Landing Zone*.
   - Tratamento de valores nulos, deduplicação e conversão de tipos de dados usando PySpark/Delta Lake.
2. **Modelagem Multidimensional (Camada Gold):**
   - Criação de tabelas Fato e Dimensão relacionando os dados atuais e históricos de clima.
3. **Analytics & Visualização (Power BI):**
   - Conexão via DirectQuery/Import para elaboração de dashboards interativos de análise climática.
