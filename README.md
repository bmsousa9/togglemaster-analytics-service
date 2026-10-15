# analytics-service (Python)

Este é o serviço de análise (analytics) do projeto ToggleMaster. Ele é um *worker* de backend e não possui API pública (exceto `/health`). Sua função é consumir os eventos da fila **AWS SQS** (preenchida pelo `evaluation-service`) e gravá-los em uma tabela do **AWS DynamoDB**.

**IMPORTANTE:** Este serviço precisa de credenciais da AWS e de uma fila SQS e uma tabela DynamoDB já criadas. Sem elas, ele encerra na partida.

## 📦 Pré-requisitos (Local)

* [Docker](https://www.docker.com/) (Docker Desktop no Windows/Mac)
* [AWS CLI](https://aws.amazon.com/cli/) configurado com as suas credenciais
* **Credenciais da AWS:** `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` e, no AWS Academy, `AWS_SESSION_TOKEN` (elas expiram a cada sessão)
* **Recursos da AWS:** uma fila SQS (Standard) e uma tabela DynamoDB

## 🗄️ Preparando a Fila e o DynamoDB

**Fila SQS** (anote a `QueueUrl` retornada):

```
aws sqs create-queue --queue-name togglemaster-events
```

**Tabela DynamoDB:** nome `ToggleMasterAnalytics`, chave primária `event_id` (String):

```
aws dynamodb create-table \
    --table-name ToggleMasterAnalytics \
    --attribute-definitions AttributeName=event_id,AttributeType=S \
    --key-schema AttributeName=event_id,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST
```

## 🚀 Rodando com Docker

**1. Clone o repositório** e entre na pasta `analytics-service`.

**2. Configure as variáveis de ambiente.** Crie um arquivo `.env` dentro da pasta `analytics-service-main`, sem aspas e sem espaço depois do `=`:

```
PORT=8005
AWS_REGION=us-east-1
AWS_SQS_URL=URL_DA_SUA_FILA
AWS_DYNAMODB_TABLE=ToggleMasterAnalytics
AWS_ACCESS_KEY_ID=SUA_CHAVE
AWS_SECRET_ACCESS_KEY=SUA_CHAVE_SECRETA
AWS_SESSION_TOKEN=SEU_TOKEN
```

**Nunca envie o `.env` para o GitHub.** Ele já está no `.gitignore`.

**3. Monte a imagem:**

```
docker build -t analytics-service ./analytics-service-main
```

**4. Inicie o container:**

```
docker run --rm -p 8005:8005 --env-file analytics-service-main/.env analytics-service
```

Ou, com o Docker Compose:

```
docker compose up --build
```

O serviço estará em `http://localhost:8005`. No terminal você verá `Clientes Boto3 inicializados` e `Iniciando o worker SQS...`.

## 🧪 Testando o Serviço

Este serviço não tem API para chamar. O teste é enviar mensagens para a fila e ver os dados chegarem ao DynamoDB.

**1. Verifique a Saúde:**

```
curl http://localhost:8005/health
```

Saída esperada: `{"status":"ok"}`

**2. Gere Eventos.** Envie uma mensagem de teste para a fila (ou faça requisições ao `evaluation-service`):

```
aws sqs send-message \
    --queue-url URL_DA_SUA_FILA \
    --message-body '{"user_id":"teste","flag_name":"enable-new-dashboard","result":true,"timestamp":"2026-10-15T12:00:00Z"}'
```

No PowerShell, grave o JSON em um arquivo `msg.json` e use `--message-body file://msg.json`.

**3. Observe os Logs.** No terminal do container, você deve ver:

`Recebidas 1 mensagens.` e `Evento ... (Flag: enable-new-dashboard) salvo no DynamoDB.`

**4. Verifique o DynamoDB:**

```
aws dynamodb scan --table-name ToggleMasterAnalytics --max-items 5
```

Saída esperada: os itens recém-criados, com `event_id`, `user_id`, `flag_name`, `result` e `timestamp`.
