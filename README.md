# Aticurando Deploy

Repositório responsável pela configuração de deploy da aplicação **Aticurando**.

O código-fonte do frontend e do backend permanece em repositórios separados. Este repositório utiliza as imagens Docker publicadas no GitHub Container Registry (GHCR) para subir toda a aplicação através do Docker Compose.

## Arquitetura

```text
Navegador
    |
    v
Frontend / Nginx
localhost:5173
    |
    | /aticurando/*
    v
Backend / API
api:3002
    |
    v
MongoDB
mongo:27017
```

As imagens utilizadas são:

```text
Frontend
ghcr.io/komura28/aticurando-frontend:latest

Backend
ghcr.io/matos795/aticurando-api:latest

MongoDB
mongo:8.0
```

## Estrutura

```text
Aticurando-Deploy/
├── compose.yaml
├── .env.example
├── .gitignore
└── README.md
```

## Pré-requisitos

É necessário ter o Docker instalado e em execução.

Para verificar:

```bash
docker --version
docker compose version
```

## Configuração

Clone o repositório:

```bash
git clone https://github.com/komura28/Aticurando-Deploy.git
cd Aticurando-Deploy
```

Crie o arquivo `.env` utilizando `.env.example` como modelo.

No PowerShell:

```powershell
Copy-Item .env.example .env
```

No Linux/macOS:

```bash
cp .env.example .env
```

Depois, configure no `.env` os valores necessários para execução da API.

O arquivo `.env` não deve ser enviado ao GitHub, pois pode conter informações sensíveis.

## Baixando as imagens

Execute:

```bash
docker compose pull
```

O Docker buscará as imagens do frontend e backend no GHCR e a imagem do MongoDB no Docker Hub.

## Iniciando a aplicação

Execute:

```bash
docker compose up -d
```

Verifique o estado dos containers:

```bash
docker compose ps
```

Os serviços devem ficar em execução e saudáveis.

A aplicação ficará disponível em:

```text
http://localhost:5173
```

## Visualizando logs

Logs de todos os serviços:

```bash
docker compose logs -f
```

Logs apenas da API:

```bash
docker compose logs -f api
```

Logs apenas do frontend:

```bash
docker compose logs -f frontend
```

Logs apenas do MongoDB:

```bash
docker compose logs -f mongo
```

## Parando a aplicação

```bash
docker compose down
```

Esse comando remove os containers e a rede criada pelo Compose, mas mantém os dados armazenados no volume do MongoDB.

Para remover também o volume e os dados do banco:

```bash
docker compose down -v
```

Use `-v` com cuidado, pois os dados armazenados no MongoDB serão apagados.

## Atualizando a aplicação

Quando novas imagens forem publicadas pelo CI/CD:

```bash
docker compose pull
docker compose up -d
```

O Docker baixará as versões mais recentes e recriará os containers quando necessário.

## Fluxo de entrega

```text
Frontend Repository
        |
        v
GitHub Actions
        |
        v
GHCR - aticurando-frontend
        |
        |
        v
Aticurando-Deploy
        ^
        |
        |
GHCR - aticurando-api
        ^
        |
GitHub Actions
        ^
        |
Backend Repository
```

O frontend e o backend possuem pipelines independentes. Após alterações aprovadas e integradas à branch `main`, novas imagens Docker são geradas e publicadas automaticamente no GHCR.

O repositório `Aticurando-Deploy` é responsável por definir como essas imagens trabalham juntas.