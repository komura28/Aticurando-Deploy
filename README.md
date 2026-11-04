# Aticurando Deploy

Repositório responsável pelas configurações de **containerização, distribuição e orquestração** da aplicação Aticurando.

O frontend e o backend permanecem em repositórios independentes. Cada aplicação gera sua própria imagem Docker através do GitHub Actions e publica essa imagem no GitHub Container Registry (GHCR).

Este repositório contém as configurações utilizadas para executar essas imagens através de:

- Docker Compose, para execução conjunta dos containers;
- Kubernetes, para orquestração, escalabilidade, self-healing, configuração e persistência.

---

# Arquitetura

A aplicação é composta por três componentes principais:

```text
Usuário
   |
   v
Frontend
React + Nginx
   |
   | /aticurando/*
   v
Backend
Node.js + Express
   |
   v
MongoDB
```

As imagens utilizadas são:

```text
Frontend
ghcr.io/komura28/aticurando-frontend:latest

Backend
ghcr.io/matos795/aticurando-api:latest

Banco de dados
mongo:8.0
```

---

# Fluxo de entrega

Frontend e backend possuem pipelines independentes.

```text
Código fonte
     |
     v
GitHub
     |
     v
GitHub Actions
     |
     | npm ci
     | build
     | Docker build
     v
GitHub Container Registry
     |
     v
Imagem Docker
```

No backend:

```text
ghcr.io/matos795/aticurando-api
```

No frontend:

```text
ghcr.io/komura28/aticurando-frontend
```

As imagens são versionadas com:

```text
latest
```

e também com o SHA do commit responsável pelo build.

Isso permite relacionar uma versão da imagem Docker diretamente com uma versão do código-fonte.

---

# Docker

## O que é Docker?

Docker permite empacotar uma aplicação junto com suas dependências em uma **imagem**.

Uma imagem é um artefato imutável utilizado para criar containers.

```text
Dockerfile
    |
    v
Docker Image
    |
    v
Container
```

No Aticurando existem imagens independentes para frontend e backend.

O MongoDB utiliza a imagem oficial:

```text
mongo:8.0
```

---

## Dockerfile

O `Dockerfile` descreve como uma imagem deve ser construída.

No backend é utilizado um build multi-stage:

```text
Node.js
   |
   | npm ci
   | TypeScript build
   v
dist/
   |
   v
Imagem final Node.js
```

A imagem final executa a aplicação utilizando um usuário não-root.

No frontend:

```text
Node.js
   |
   | npm ci
   | Vite build
   v
dist/
   |
   v
Nginx
```

O Nginx serve os arquivos estáticos do React e também funciona como proxy para a API.

---

# Docker Compose

Docker Compose permite definir e executar vários containers relacionados.

No Aticurando:

```text
Docker Compose
│
├── frontend
│
├── api
│
└── mongo
```

O Compose cria uma rede própria e permite que os containers utilizem o nome dos serviços como endereço.

Por exemplo:

```text
frontend → api:3002
api → mongo:27017
```

---

# Executando com Docker Compose

## Pré-requisitos

É necessário possuir Docker instalado e em execução.

```bash
docker --version
docker compose version
```

Clone o repositório:

```bash
git clone https://github.com/komura28/Aticurando-Deploy.git
cd Aticurando-Deploy
```

Crie o `.env` utilizando `.env.example`.

No PowerShell:

```powershell
Copy-Item .env.example .env
```

Linux/macOS:

```bash
cp .env.example .env
```

Configure os valores necessários no `.env`.

O `.env` não deve ser enviado ao GitHub.

---

## Baixar as imagens

```bash
docker compose pull
```

As imagens serão obtidas de:

```text
Frontend → GitHub Container Registry
Backend  → GitHub Container Registry
MongoDB  → Docker Hub
```

---

## Iniciar

```bash
docker compose up -d
```

Verifique:

```bash
docker compose ps
```

A aplicação estará disponível em:

```text
http://localhost:5173
```

---

## Logs

```bash
docker compose logs -f
```

Ou individualmente:

```bash
docker compose logs -f frontend
docker compose logs -f api
docker compose logs -f mongo
```

---

## Parar

```bash
docker compose down
```

O volume do banco permanece existente.

Para remover também o volume:

```bash
docker compose down -v
```

Esse comando remove os dados persistidos pelo MongoDB.

---

# Kubernetes

## Por que Kubernetes?

Docker é responsável por criar e executar containers.

Docker Compose permite organizar vários containers, principalmente em uma única máquina.

Kubernetes adiciona uma camada de **orquestração**.

Em vez de apenas iniciar containers, declaramos qual deve ser o estado da aplicação.

Por exemplo:

```text
Estado desejado:
3 APIs funcionando
```

Se uma API falhar:

```text
Estado atual:
2 APIs funcionando
```

Kubernetes detecta a diferença e cria outra instância.

```text
Estado desejado: 3
Estado atual:     3
```

Esse conceito é chamado de **estado desejado**.

---

# Arquitetura Kubernetes do Aticurando

```text
                    Kubernetes Cluster
                           |
                    Namespace aticurando
                           |
          +----------------+----------------+
          |                                 |
          v                                 v
Frontend Deployment                 API Deployment
1 réplica                           3 réplicas
          |                                 |
          v                                 v
Frontend Pod                     API Pods
          |                         | | |
          v                         v v v
Frontend Service                API Service
                                      |
                                      v
                               Mongo Service
                                      |
                                      v
                              Mongo StatefulSet
                                      |
                                      v
                                  mongo-0
                                      |
                                      v
                                      PVC
```

---

# Conceitos Kubernetes utilizados

## Cluster

É o ambiente Kubernetes completo.

Durante o desenvolvimento utilizamos um cluster local criado com **kind — Kubernetes IN Docker**.

```bash
kind create cluster --name aticurando
```

---

## Node

Node é uma máquina pertencente ao cluster e responsável por executar workloads.

No ambiente local:

```bash
kubectl get nodes
```

Exemplo:

```text
aticurando-control-plane   Ready
```

---

## Namespace

Namespaces permitem separar e organizar recursos dentro do cluster.

O projeto utiliza:

```text
aticurando
```

Criado através de:

```yaml
kind: Namespace
metadata:
  name: aticurando
```

---

## Pod

Pod é a menor unidade gerenciada pelo Kubernetes.

Normalmente, no Aticurando:

```text
Pod
└── Container
```

Os containers do frontend, backend e Mongo executam dentro de Pods.

Pods são considerados descartáveis e podem ser substituídos pelo Kubernetes.

---

## Deployment

Deployment mantém uma quantidade desejada de Pods.

Frontend:

```text
replicas: 1
```

Backend:

```text
replicas: 3
```

Arquitetura da API:

```text
Deployment API
│
├── API Pod 1
├── API Pod 2
└── API Pod 3
```

Se um Pod for removido, o Deployment cria outro automaticamente.

Esse comportamento demonstra o conceito de **self-healing**.

---

## Service

Pods podem ser criados e destruídos e seus endereços podem mudar.

Services fornecem um endereço estável para comunicação.

No Aticurando:

```text
Frontend
   |
   v
Service api
   |
   +--> API Pod 1
   +--> API Pod 2
   +--> API Pod 3
```

A aplicação frontend não precisa conhecer os IPs dos Pods.

Ela acessa simplesmente:

```text
api:3002
```

Da mesma forma, a API acessa o Mongo através de:

```text
mongo:27017
```

---

## ConfigMap

ConfigMap armazena configurações não sensíveis.

Exemplos utilizados pela API:

```text
PORT
MONGO_URI
JWT_EXPIRES_IN
FRONTEND_URL
PASSWORD_RESET_EXPIRES_MINUTES
```

Arquivo:

```text
k8s/api-configmap.yaml
```

---

## Secret

Secret é utilizado para separar informações sensíveis das configurações comuns.

Exemplos:

```text
JWT_SECRET
USER_ADMIN
USER_ADMIN_PASSWORD
BREVO_API_KEY
BREVO_SENDER_EMAIL
```

Os valores reais não são enviados para o GitHub.

O arquivo local utilizado para criação do Secret é:

```text
k8s/secrets.env
```

e está incluído no `.gitignore`.

---

## Readiness Probe

Readiness responde:

```text
"Este Pod está pronto para receber requisições?"
```

Na API é utilizado:

```text
GET /aticurando/v1/health
```

Esse endpoint também verifica a conexão com o MongoDB.

Se o banco estiver indisponível:

```text
API continua executando
       |
       v
Readiness falha
       |
       v
Pod deixa de receber tráfego
```

---

## Liveness Probe

Liveness responde:

```text
"Este container ainda está funcionando?"
```

Caso o teste falhe repetidamente, Kubernetes pode reiniciar o container.

Na API é verificado se existe um processo respondendo na porta:

```text
3002
```

No frontend, o Nginx disponibiliza:

```text
/healthz
```

---

# MongoDB e StatefulSet

Frontend e API são aplicações stateless.

Por isso utilizam:

```text
Deployment
```

MongoDB possui dados que precisam permanecer entre recriações dos Pods.

Por isso utiliza:

```text
StatefulSet
```

O Pod do Mongo possui identidade estável:

```text
mongo-0
```

---

# Persistência

Pods são descartáveis.

Os dados do banco não podem depender do filesystem do Pod.

Por isso utilizamos um:

```text
PersistentVolumeClaim
```

ou:

```text
PVC
```

O fluxo é:

```text
Mongo
  |
  v
/data/db
  |
  v
PVC mongo-data
  |
  v
PersistentVolume
```

Mesmo que:

```text
mongo-0
```

seja removido e recriado, o novo Pod monta novamente o mesmo armazenamento.

Isso permite que os dados permaneçam existentes.

---

# Estrutura do repositório

```text
Aticurando-Deploy/
│
├── compose.yaml
├── .env.example
├── .gitignore
├── README.md
│
└── k8s/
    ├── namespace.yaml
    │
    ├── frontend-deployment.yaml
    ├── frontend-service.yaml
    │
    ├── api-configmap.yaml
    ├── api-deployment.yaml
    ├── api-service.yaml
    │
    ├── mongo-statefulset.yaml
    ├── mongo-service.yaml
    ├── mongo-headless-service.yaml
    └── mongo-pvc.yaml
```

O arquivo:

```text
k8s/secrets.env
```

existe apenas localmente e não deve ser versionado.

---

# Executando com Kubernetes

## Pré-requisitos

São necessários:

```text
Docker
kubectl
kind
```

Verifique:

```bash
docker --version
kubectl version --client
kind version
```

---

## Criando o cluster

```bash
kind create cluster --name aticurando
```

Confira:

```bash
kubectl get nodes
```

---

## Criando o namespace

```bash
kubectl apply -f k8s/namespace.yaml
```

---

## Criando o Secret

Crie localmente:

```text
k8s/secrets.env
```

com as variáveis sensíveis necessárias.

No PowerShell:

```powershell
kubectl create secret generic api-secret `
  --from-env-file=k8s/secrets.env `
  -n aticurando
```

---

## Aplicando o MongoDB

```bash
kubectl apply -f k8s/mongo-pvc.yaml
kubectl apply -f k8s/mongo-service.yaml
kubectl apply -f k8s/mongo-headless-service.yaml
kubectl apply -f k8s/mongo-statefulset.yaml
```

---

## Aplicando a API

```bash
kubectl apply -f k8s/api-configmap.yaml
kubectl apply -f k8s/api-service.yaml
kubectl apply -f k8s/api-deployment.yaml
```

---

## Aplicando o frontend

```bash
kubectl apply -f k8s/frontend-service.yaml
kubectl apply -f k8s/frontend-deployment.yaml
```

---

# Verificando o cluster

```bash
kubectl get pods -n aticurando
```

Exemplo:

```text
api-xxxxx        1/1   Running
api-yyyyy        1/1   Running
api-zzzzz        1/1   Running
frontend-xxxxx   1/1   Running
mongo-0          1/1   Running
```

Deployments:

```bash
kubectl get deployments -n aticurando
```

StatefulSets:

```bash
kubectl get statefulsets -n aticurando
```

Services:

```bash
kubectl get services -n aticurando
```

Persistência:

```bash
kubectl get pvc -n aticurando
```

---

# Acessando a aplicação

No ambiente local utilizamos `port-forward`:

```bash
kubectl port-forward service/frontend 5173:8080 -n aticurando
```

Acesse:

```text
http://localhost:5173
```

O fluxo completo será:

```text
Navegador
    |
    v
localhost:5173
    |
    v
Frontend Service
    |
    v
Frontend Pod
    |
    v
API Service
    |
    +--> API Pod 1
    +--> API Pod 2
    +--> API Pod 3
            |
            v
       Mongo Service
            |
            v
          mongo-0
            |
            v
        PVC mongo-data
```

---

# Demonstrações realizadas

## Self-healing

Um Pod da API pode ser removido manualmente:

```bash
kubectl delete pod NOME_DO_POD -n aticurando
```

O Deployment detecta que existem menos réplicas do que o estado desejado e cria automaticamente outro Pod.

---

## Escalabilidade

A API utiliza:

```yaml
replicas: 3
```

O Service permite que todas as réplicas sejam acessadas através de um único endereço:

```text
api:3002
```

---

## Health checks

Quando o MongoDB é temporariamente indisponibilizado, a readiness da API falha.

A API continua executando, mas deixa de ser considerada pronta para receber tráfego.

Quando o banco volta, a API automaticamente volta ao estado `Ready`.

---

## Persistência

Após criar dados no sistema, o Pod:

```text
mongo-0
```

foi removido.

O StatefulSet recriou o Pod utilizando novamente o mesmo PVC.

Os dados permaneceram disponíveis, comprovando que o armazenamento não dependia do ciclo de vida do Pod.

---

# Docker x Kubernetes no Aticurando

Docker resolveu o problema de **empacotar e executar** a aplicação.

```text
Código
  ↓
Docker Image
  ↓
Container
```

Docker Compose resolveu a execução conjunta dos containers:

```text
Frontend
API
MongoDB
```

Kubernetes adicionou a camada de **orquestração**:

```text
Containers
    |
    v
Pods
    |
    v
Deployments / StatefulSet
    |
    v
Services
    |
    +--> Self-healing
    +--> Escalabilidade
    +--> Service discovery
    +--> Health checks
    +--> Configuração
    +--> Secrets
    └--> Persistência
```

Para o tamanho atual do Aticurando, Docker Compose já seria suficiente para executar a aplicação em um único servidor.

A implementação de Kubernetes foi utilizada para estudar e demonstrar conceitos de orquestração utilizados em ambientes maiores, distribuídos e com maiores requisitos de disponibilidade.

---

# Resumo da arquitetura

```text
GitHub
│
├── Backend
│     ↓
│ GitHub Actions
│     ↓
│ GHCR
│
├── Frontend
│     ↓
│ GitHub Actions
│     ↓
│ GHCR
│
└── Aticurando-Deploy
        |
        v
   Kubernetes
        |
        ├── Frontend Deployment
        ├── Frontend Service
        │
        ├── API Deployment (3 réplicas)
        ├── API Service
        ├── ConfigMap
        ├── Secret
        │
        ├── Mongo StatefulSet
        ├── Mongo Services
        └── PVC
```

O resultado é uma aplicação containerizada, distribuída através de registry e orquestrada através do Kubernetes.