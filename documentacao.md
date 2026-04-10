# Documentacao — Operacao Ufology

## Visao geral

Este projeto implementa a infraestrutura completa da plataforma Ufology no Kubernetes, junto com pipelines de CI/CD usando GitHub Actions. A ideia geral eh subir um banco PostgreSQL, um cache Redis e uma aplicacao Spring Boot que se conecta a ambos, tudo dentro de um namespace isolado no cluster.

## Estrutura do projeto

```
.
├── k8s-manifests.yaml          # manifests kubernetes (missoes 1, 2 e 4)
├── ufoTracker/                 # codigo-fonte da aplicacao
│   ├── Dockerfile              # dockerfile criado na missao 3
│   ├── pom.xml
│   ├── mvnw
│   └── src/
├── .github/workflows/
│   ├── hello.yml               # parte 2.1 - hello ci/cd
│   ├── tests.yml               # parte 2.2 - testes em pull request
│   ├── gradle-ci.yml           # parte 2.3 - build com maven
│   ├── env-demo.yml            # parte 3.1 - variavel de ambiente
│   └── secret-demo.yml         # parte 3.2 - uso de secrets
└── documentacao.md             # este arquivo
```

## Parte 1 — Kubernetes

### Missao 1 — PostgreSQL

Criei um namespace chamado `ufology` para isolar todos os recursos da operacao. O deployment do PostgreSQL usa a imagem customizada `leogloriainfnet/ufodb:1.0-win` (tag amd64, compativel com o servidor de deploy). As variaveis de ambiente `POSTGRES_USER`, `POSTGRES_PASSWORD` e `POSTGRES_DB` sao definidas diretamente no manifesto, conforme pedido. O service `postgres-svc` expoe a porta 5432 internamente via ClusterIP.

### Missao 2 — Redis

O Redis usa a imagem `redis:alpine`, que eh bem leve. Segue a mesma logica: um deployment com 1 replica e um service `redis-svc` na porta 6379. Tudo no namespace `ufology`.

### Missao 3 — Dockerizacao da aplicacao

O ufoTracker eh uma aplicacao Spring Boot com Java 21 e Maven. Criei um Dockerfile multi-stage:

1. **Stage de build**: usa `eclipse-temurin:21-jdk-alpine`, copia os arquivos do Maven primeiro (pra cachear dependencias), depois copia o source e roda `mvnw package`.
2. **Stage final**: usa `eclipse-temurin:21-jre-alpine` (so o JRE, imagem menor), copia o jar gerado e expoe a porta 8080.

Comandos pra build e push da imagem:

```bash
cd ufoTracker
docker build -t vctorgriggi/ufo-tracker:latest .
docker push vctorgriggi/ufo-tracker:latest
```

Link publico da imagem: `https://hub.docker.com/r/vctorgriggi/ufo-tracker`

### Missao 4 — Deploy da aplicacao no cluster

Aqui a ideia eh conectar tudo. Criei:

- **ConfigMap `app-config`**: guarda `DB_NAME=ufology` (dado nao-sensivel).
- **Secret `db-secret`**: guarda `DB_PASSWORD=devops2025!` em base64 (dado sensivel).
- **Deployment `ufo-tracker`**: 2 replicas usando a imagem publicada no Docker Hub. As variaveis de ambiente sobrescrevem as configuracoes do `application.yaml` pra apontar pros services do cluster em vez de `localhost`:
  - `SPRING_DATASOURCE_URL` → aponta pra `postgres-svc:5432`
  - `SPRING_DATA_REDIS_HOST` → aponta pra `redis-svc`
  - `POSTGRES_PASSWORD` vem do Secret
- **Service `ufo-tracker-svc`**: expoe a app na porta 8080 via ClusterIP.

## Parte 2 — Workflows GitHub Actions

### hello.yml
Roda em qualquer push e simplesmente exibe "Hello CI/CD" no log. O workflow mais simples possivel.

### tests.yml
Dispara em pull requests. Faz checkout do codigo com `actions/checkout@v4` e executa `echo "Rodando testes"`.

### gradle-ci.yml
Roda em push pra branch main. Como o projeto usa Maven (nao Gradle), o enunciado orienta simular com Maven. O workflow configura Java 21 com Temurin e executa `./mvnw package -DskipTests` no diretorio do ufoTracker.

## Parte 3 — Runners, Variaveis e Seguranca

### env-demo.yml
Define `DEPLOY_ENV=staging` como variavel de ambiente no nivel do workflow e exibe o valor no log.

### secret-demo.yml
Referencia o secret `API_KEY` configurado no repositorio (via Settings > Secrets). O workflow verifica se o secret esta presente e exibe "API_KEY configurado" sem expor o valor real.

### Runners hospedados vs auto-hospedados

**Runners hospedados pelo GitHub** sao maquinas virtuais gerenciadas pelo proprio GitHub. Vantagens: nao precisa configurar nada, ja vem com diversas ferramentas instaladas, escalam automaticamente e sao mantidos pelo GitHub. Desvantagens: tem limites de uso (minutos por mes dependendo do plano), nao da pra personalizar muito o ambiente, e nao tem acesso a recursos da rede interna da empresa.

**Runners auto-hospedados (self-hosted)** sao maquinas que voce mesmo configura e registra no GitHub. Vantagens: controle total sobre o hardware e software, acesso a recursos internos da rede, sem limite de minutos, e da pra usar maquinas com specs especificos (GPU, muito RAM, etc). Desvantagens: voce eh responsavel pela manutencao, seguranca e atualizacao do runner, alem de ter que gerenciar a infraestrutura.

Na pratica, pra maioria dos projetos os runners hospedados resolvem bem. Runners self-hosted fazem mais sentido quando voce precisa acessar algo interno ou tem requisitos especificos de hardware.

## Como rodar

### Pre-requisitos
- Docker instalado
- Kubernetes rodando (minikube, kind, Docker Desktop, etc)
- kubectl configurado
- Conta no Docker Hub

### Passo a passo

1. **Aplicar os manifests do Kubernetes**:
```bash
kubectl apply -f k8s-manifests.yaml
```

2. **Verificar se tudo subiu**:
```bash
kubectl get all -n ufology
```

3. **Build e push da imagem da aplicacao** (missao 3):
```bash
cd ufoTracker
docker build -t vctorgriggi/ufo-tracker:latest .
docker push vctorgriggi/ufo-tracker:latest
```

4. **Verificar os pods**:
```bash
kubectl get pods -n ufology
```

5. **Para os workflows do GitHub Actions**: basta dar push pro repositorio remoto. O `hello.yml`, `env-demo.yml` e `secret-demo.yml` disparam automaticamente em qualquer push. O `gradle-ci.yml` dispara em push pra main. O `tests.yml` dispara ao abrir um pull request.

6. **Configurar o secret `API_KEY`** no GitHub: va em Settings > Secrets and variables > Actions > New repository secret, crie um secret chamado `API_KEY` com qualquer valor.

### Observacao sobre a tag da imagem PostgreSQL

No manifesto, estou usando `leogloriainfnet/ufodb:1.0-win` (amd64) pois o deploy roda em servidor Linux. Se for rodar localmente num Mac Apple Silicon, troque para `leogloriainfnet/ufodb:1.0-mac`.
