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

## ☁️ Publicando a imagem no AWS ECR (conta da equipe)

Na conta compartilhada do grupo, a fila SQS é a mesma do `evaluation-service` (`evaluation-events`), então não é preciso criá-la. Use a URL dela em `AWS_SQS_URL`.

**1. Monte a imagem com a tag `v1`:**

```
docker build -t analytics-service:v1 ./analytics-service-main
```

**2. Crie o repositório no ECR** (uma vez só):

```
aws ecr create-repository --repository-name analytics-service --region us-east-1
```

**3. Faça login, nomeie e envie a imagem.** Troque `ACCOUNT_ID` pelo ID da conta AWS:

```
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
docker tag analytics-service:v1 ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/analytics-service:v1
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/analytics-service:v1
```

**4. Confira que a tag `v1` chegou:**

```
aws ecr describe-images --repository-name analytics-service --region us-east-1 --query "imageDetails[].imageTags"
```

Saída esperada: `["v1"]`.

> Se usar um perfil do AWS CLI diferente do padrão, acrescente `--profile NOME_DO_PERFIL` aos comandos `aws`.

## ☸️ Deploy no EKS

Os manifestos ficam na pasta `k8s/` e usam o namespace `microservices`, compartilhado com os outros serviços:

| Arquivo | O que faz |
|---|---|
| `01-configmap.yaml` | Porta, região, URL da fila SQS e nome da tabela DynamoDB |
| `02-secret.yaml.example` | Modelo do Secret com as credenciais da AWS (não é aplicado) |
| `03-deployment.yaml` | Pod do serviço, com `requests`, `limits` e probes em `/health` |
| `04-service.yaml` | Service `ClusterIP` na porta 8005 |
| `05-hpa.yaml` | HPA por CPU (alvo de 70%, de 1 a 5 pods) |

**1. Conecte o `kubectl` ao cluster:**

```
aws eks update-kubeconfig --region us-east-1 --name microservices-cluster
```

**2. Crie o Secret com as credenciais da AWS.** O serviço lê a SQS e grava no DynamoDB, e a role dos nós não dá essa permissão ao pod. O comando lê as credenciais do seu perfil e não as mostra na tela (troque `NOME_DO_PERFIL`):

```
$ak = aws configure get aws_access_key_id --profile NOME_DO_PERFIL
$sk = aws configure get aws_secret_access_key --profile NOME_DO_PERFIL
$st = aws configure get aws_session_token --profile NOME_DO_PERFIL
kubectl create secret generic analytics-service-secret -n microservices "--from-literal=AWS_ACCESS_KEY_ID=$ak" "--from-literal=AWS_SECRET_ACCESS_KEY=$sk" "--from-literal=AWS_SESSION_TOKEN=$st"
Remove-Variable ak,sk,st
```

No AWS Academy as credenciais expiram a cada sessão. Quando renovar o laboratório, apague o Secret (`kubectl delete secret analytics-service-secret -n microservices`), crie de novo e reinicie o Deployment (`kubectl rollout restart deployment/analytics-service -n microservices`).

**3. Aplique os manifestos:**

```
kubectl apply -f k8s/
```

**4. Confira:**

```
kubectl get pods -n microservices
kubectl get hpa -n microservices
kubectl logs deploy/analytics-service -n microservices
```

Nos logs aparecem `Found credentials in environment variables` e `Iniciando o worker SQS...`. Ao chegar uma mensagem na fila, aparece `Evento ... salvo no DynamoDB.`

**Escalabilidade:** o HPA usa a CPU porque, no AWS Academy, ferramentas como o KEDA não funcionam (não é possível criar roles do IAM). Quando a fila enche, o serviço processa mais mensagens, a CPU sobe e o HPA adiciona pods.

> Na conta do grupo, a fila é a `evaluation-events` e a tabela do DynamoDB também se chama `evaluation-events` (chave `event_id`, tipo texto).
