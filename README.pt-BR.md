# Case de MLOps para Previsão de Churn

[Read in English](README.md)

Workflow de machine learning reproduzível para classificação de churn de clientes. Este case demonstra preparação de dados, treinamento de modelo, versionamento de artefatos e rastreabilidade de experimentos com Git e DVC.

## Competências demonstradas

- Preparação de dados e divisão treino/teste determinística em Python.
- Classificação com Random Forest e scikit-learn.
- Versionamento de dados e modelos com [DVC](https://dvc.org/), sem enviar arquivos grandes ao Git.
- Pipeline orientada a dependências em `dvc.yaml` e hashes de execução em `dvc.lock`.
- Reprodutibilidade, colaboração e rollback com Git + DVC.

## Contexto de negócio

Churn é um problema de retenção: identificar clientes com risco de saída permite ações como ofertas direcionadas, melhoria de serviço e suporte proativo. O projeto transforma um CSV bruto em dados preparados, modelo treinado e métrica rastreada por meio de uma pipeline reproduzível.

## Pipeline

```text
data/raw/churn.csv  ──► prepare ──► data/processed/{train,test}.csv
                                │
                                ▼
                           train ──► models/model.pkl
                                 └─► metrics.json
```

| Estágio | Script | Entradas | Saídas |
| --- | --- | --- | --- |
| `prepare` | `src/prepare.py` | CSV bruto de churn | CSVs de treino e teste |
| `train` | `src/train.py` | Dados preparados | Modelo serializado e métrica |

O grafo está em [`dvc.yaml`](dvc.yaml) e [`dvc.lock`](dvc.lock) registra os hashes exatos da última execução.

## Implementação atual

`src/prepare.py` lê `data/raw/churn.csv`, preenche valores ausentes de colunas numéricas pela mediana e gera divisão 80/20 com `random_state=42`. A seed fixa torna o resultado repetível.

`src/train.py` treina `RandomForestClassifier` com `n_estimators=50` e `random_state=42`, grava `models/model.pkl` e salva a acurácia em `metrics.json`.

A métrica atual é **1.0 de acurácia de treino**. Ela comprova a execução da pipeline, mas não é uma estimativa de produção: a avaliação utiliza os mesmos dados do treino. A próxima evolução deve avaliar `data/processed/test.csv` com accuracy, precision, recall, F1, ROC-AUC e matriz de confusão.

## Início rápido

### Pré-requisitos

- Python 3.12+
- Git
- Remote DVC e credenciais, quando os dados brutos não estiverem disponíveis localmente

Ambientes virtuais são ignorados pelo Git; cada pessoa desenvolvedora cria o seu.

**Windows PowerShell**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

**macOS/Linux**

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### Obter os dados e executar

O Git guarda o ponteiro [`data/raw/churn.csv.dvc`](data/raw/churn.csv.dvc), não o CSV original. Com remote configurado:

```bash
dvc pull
dvc repro
```

Sem remote, coloque o arquivo autorizado em `data/raw/churn.csv` e registre-o:

```bash
dvc add data/raw/churn.csv
git add data/raw/churn.csv.dvc data/raw/.gitignore
dvc repro
```

O DVC executa `prepare` antes de `train` e atualiza `dvc.lock`. Sem mudanças, uma nova execução ignora os estágios; ao alterar apenas `src/train.py`, somente o treino é reexecutado.

```bash
dvc status
dvc dag
dvc metrics show
dvc metrics diff HEAD~1
```

## Versionamento híbrido: Git + DVC

| Git versiona | DVC versiona |
| --- | --- |
| Código, `dvc.yaml`, `dvc.lock`, configuração e documentação | Dados brutos/processados, modelos e outros artefatos grandes |
| Ponteiros leves, como `churn.csv.dvc` | Hashes de conteúdo e objetos no cache/remote |
| Histórico revisável e colaboração | Linhagem e reprodução de artefatos |

Os `.gitignore` gerados pelo DVC evitam que o CSV bruto, `data/processed/`, `models/model.pkl` e `metrics.json` sejam enviados ao Git como arquivos comuns. O Git recebe pipeline e metadados; o DVC armazena os artefatos no cache e no armazenamento remoto.

## Fluxo para times

1. Execute `git pull` e depois `dvc pull`.
2. Altere código, parâmetros ou dados.
3. Execute `dvc repro` e revise `dvc metrics show`.
4. Faça commit dos metadados de reprodução:

   ```bash
   git add src dvc.yaml dvc.lock requirements.txt README.md README.pt-BR.md
   git commit -m "exp: improve churn model"
   ```

5. Execute `dvc push` para publicar artefatos e envie o commit Git.

Para recuperar experimento anterior:

```bash
git checkout <commit-ou-tag>
dvc checkout
```

Execute `dvc pull` antes quando os artefatos precisarem ser recuperados do remote.

## Remote DVC e segurança

Use armazenamento com controle de acesso, como S3, Azure Blob, Google Drive, SSH ou filesystem compartilhado. Exemplo local apenas para demonstração:

```bash
dvc remote add -d storage ../dvc-storage
dvc push
```

Credenciais ficam fora do Git. Nunca versione segredos ou dados sensíveis de clientes.

## Estrutura do repositório

```text
data/raw/          # Ponteiro DVC e CSV bruto ignorado
data/processed/    # Saída da pipeline; Git ignora, DVC gerencia
models/            # Saída do modelo; Git ignora, DVC gerencia
src/prepare.py     # Limpeza e divisão determinística
src/train.py       # Treino e geração da métrica
dvc.yaml           # DAG da pipeline
dvc.lock           # Hashes para reprodução
requirements.txt   # Dependências Python
case.txt           # Briefing original
```

## Próximos passos de MLOps

- Mover configurações para `params.yaml` e usar `dvc exp run`.
- Avaliar no conjunto de teste com métricas orientadas ao negócio.
- Adicionar validação de atributos, testes de dados e prevenção de leakage.
- Criar CI para lint, testes e reprodução controlada da pipeline.
- Adicionar registro de modelos, deploy, monitoramento, drift e critérios de retreinamento.
- Documentar proveniência, consentimento, retenção, acesso e fairness antes de uso operacional.

## Para quem revisa este case

O objetivo é tornar decisões de engenharia visíveis: metadados leves no Git, artefatos grandes no DVC, DAG declarativa, execução determinística e lockfile para rastreabilidade. Esses padrões facilitam revisar, reproduzir, comparar e reverter experimentos com segurança.
