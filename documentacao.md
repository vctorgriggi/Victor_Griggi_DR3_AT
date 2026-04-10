# Documentação — Operação Ufology

## Visão geral

Este projeto implementa a infraestrutura completa da plataforma Ufology no Kubernetes, junto com pipelines de CI/CD usando GitHub Actions. A ideia geral é subir um banco PostgreSQL, um cache Redis e uma aplicação Spring Boot que se conecta a ambos, tudo dentro de um namespace isolado no cluster.

## Estrutura do projeto

```
.
├── k8s-manifests.yaml          # manifests kubernetes (missões 1, 2 e 4)
├── ufoTracker/                 # código-fonte da aplicação
│   ├── Dockerfile              # dockerfile criado na missão 3
│   ├── pom.xml
│   ├── mvnw
│   └── src/
├── .github/workflows/
│   ├── hello.yml               # parte 2.1 - hello ci/cd
│   ├── tests.yml               # parte 2.2 - testes em pull request
│   ├── gradle-ci.yml           # parte 2.3 - build com maven
│   ├── env-demo.yml            # parte 3.1 - variável de ambiente
│   └── secret-demo.yml         # parte 3.2 - uso de secrets
└── documentacao.md             # este arquivo
```

## Parte 1 — Kubernetes

### Missão 1 — PostgreSQL

Criei um namespace chamado `ufology` para isolar todos os recursos da operação. O deployment do PostgreSQL usa a imagem customizada `leogloriainfnet/ufodb:1.0-win` (tag amd64, compatível com o servidor de deploy). As variáveis de ambiente `POSTGRES_USER`, `POSTGRES_PASSWORD` e `POSTGRES_DB` são definidas diretamente no manifesto, conforme pedido. O service `postgres-svc` expõe a porta 5432 internamente via ClusterIP.

### Missão 2 — Redis

O Redis usa a imagem `redis:alpine`, que é bem leve. Segue a mesma lógica: um deployment com 1 réplica e um service `redis-svc` na porta 6379. Tudo no namespace `ufology`.

### Missão 3 — Dockerização da aplicação

O ufoTracker é uma aplicação Spring Boot com Java 21 e Maven. Criei um Dockerfile multi-stage:

1. **Stage de build**: usa `eclipse-temurin:21-jdk-alpine`, copia os arquivos do Maven primeiro (para cachear dependências), depois copia o source e roda `mvnw package`.
2. **Stage final**: usa `eclipse-temurin:21-jre-alpine` (só o JRE, imagem menor), copia o jar gerado e expõe a porta 8080.

Comandos para build e push da imagem:

```bash
cd ufoTracker
docker build -t vctorgriggi/ufo-tracker:latest .
docker push vctorgriggi/ufo-tracker:latest
```

Link público da imagem: `https://hub.docker.com/r/vctorgriggi/ufo-tracker`

### Missão 4 — Deploy da aplicação no cluster

Aqui a ideia é conectar tudo. Criei:

- **ConfigMap `app-config`**: guarda `DB_NAME=ufology` (dado não-sensível).
- **Secret `db-secret`**: guarda `DB_PASSWORD=devops2025!` em base64 (dado sensível).
- **Deployment `ufo-tracker`**: 2 réplicas usando a imagem publicada no Docker Hub. As variáveis de ambiente sobrescrevem as configurações do `application.yaml` para apontar para os services do cluster em vez de `localhost`:
  - `SPRING_DATASOURCE_URL` → aponta para `postgres-svc:5432`
  - `SPRING_DATA_REDIS_HOST` → aponta para `redis-svc`
  - `POSTGRES_PASSWORD` vem do Secret
- **Service `ufo-tracker-svc`**: expõe a app na porta 8080 via ClusterIP.

## Parte 2 — Workflows GitHub Actions

### hello.yml
Roda em qualquer push e simplesmente exibe "Hello CI/CD" no log. O workflow mais simples possível.

### tests.yml
Dispara em pull requests. Faz checkout do código com `actions/checkout@v4` e executa `echo "Rodando testes"`.

### gradle-ci.yml
Roda em push para a branch main. Como o projeto usa Maven (não Gradle), o enunciado orienta simular com Maven. O workflow configura Java 21 com Temurin e executa `./mvnw package -DskipTests` no diretório do ufoTracker.

## Parte 3 — Runners, Variáveis e Segurança

### env-demo.yml
Define `DEPLOY_ENV=staging` como variável de ambiente no nível do workflow e exibe o valor no log.

### secret-demo.yml
Referencia o secret `API_KEY` configurado no repositório (via Settings > Secrets). O workflow verifica se o secret está presente e exibe "API_KEY configurado" sem expor o valor real.

### Runners hospedados vs auto-hospedados

**Runners hospedados pelo GitHub** são máquinas virtuais gerenciadas pelo próprio GitHub. Vantagens: não precisa configurar nada, já vem com diversas ferramentas instaladas, escalam automaticamente e são mantidos pelo GitHub. Desvantagens: tem limites de uso (minutos por mês dependendo do plano), não é possível personalizar muito o ambiente, e não tem acesso a recursos da rede interna da empresa.

**Runners auto-hospedados (self-hosted)** são máquinas que você mesmo configura e registra no GitHub. Vantagens: controle total sobre o hardware e software, acesso a recursos internos da rede, sem limite de minutos, e é possível usar máquinas com specs específicos (GPU, muita RAM, etc). Desvantagens: você é responsável pela manutenção, segurança e atualização do runner, além de ter que gerenciar a infraestrutura.

Na prática, para a maioria dos projetos os runners hospedados resolvem bem. Runners self-hosted fazem mais sentido quando você precisa acessar algo interno ou tem requisitos específicos de hardware.

## Como rodar

### Pré-requisitos
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

3. **Build e push da imagem da aplicação** (missão 3):
```bash
cd ufoTracker
docker build -t vctorgriggi/ufo-tracker:latest .
docker push vctorgriggi/ufo-tracker:latest
```

4. **Verificar os pods**:
```bash
kubectl get pods -n ufology
```

5. **Para os workflows do GitHub Actions**: basta dar push para o repositório remoto. O `hello.yml`, `env-demo.yml` e `secret-demo.yml` disparam automaticamente em qualquer push. O `gradle-ci.yml` dispara em push para main. O `tests.yml` dispara ao abrir um pull request.

6. **Configurar o secret `API_KEY`** no GitHub: vá em Settings > Secrets and variables > Actions > New repository secret, crie um secret chamado `API_KEY` com qualquer valor.

### Observação sobre a tag da imagem PostgreSQL

No manifesto, estou usando `leogloriainfnet/ufodb:1.0-win` (amd64) pois o deploy roda em servidor Linux. Se for rodar localmente num Mac Apple Silicon, troque para `leogloriainfnet/ufodb:1.0-mac`.
