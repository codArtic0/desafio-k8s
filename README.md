<div align="center">

# Kubernetes PostgreSQL + PostgREST

### Dados persistentes, API REST e operações Kubernetes automatizadas

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29%2B-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![PostgREST](https://img.shields.io/badge/PostgREST-12.2.3-F0654B?logo=postgresql&logoColor=white)](https://postgrest.org/)
[![CI](https://img.shields.io/badge/CI-Kubeconform%20%7C%20Gitleaks-2088FF?logo=githubactions&logoColor=white)](.github/workflows/ci.yml)

<p>
  Ambiente Kubernetes completo para um banco PostgreSQL persistente e uma API REST usando PostgREST.
</p>

</div>

<div align="center">

![Arquitetura](./assets/Diagrama.png)

</div>

## Conteúdo

- [Visão geral](#visão-geral)
- [Componentes](#componentes)
- [Fluxo de requisições](#fluxo-de-requisições)
- [Pré-requisitos](#pré-requisitos)
- [Configuração](#configuração)
- [Deploy](#deploy)
- [Testar a API](#testar-a-api)
- [Persistência](#persistência)
- [Escalonamento](#escalonamento)
- [CI/CD](#cicd)
- [Limpeza](#limpeza)
- [Documentação dos níveis](#documentação-dos-níveis)

## Estrutura do projeto

```text
.
├── assets/
│   ├── Diagrama.png
│   └── evidencias/
│       ├── nivel1/
│       ├── nivel2/
│       ├── nivel3/
│       ├── nivel4/
│       ├── nivel5/
│       ├── nivel6/
│       └── nivel7/
├── docs/
│   ├── nivel1-namespace.md
│   ├── nivel2-postgres.md
│   ├── nivel3-configuracao-segredos.md
│   ├── nivel4-api-postgrest.md
│   ├── nivel5-exposicao-persistencia.md
│   ├── nivel6-health-checks-escala.md
│   └── nivel7-escalonamento-automatico.md
├── k8s/
│   ├── 00-namespace.yaml
│   ├── 01-postgres-secret.yaml
│   ├── 01-postgres-secret.yaml.example
│   ├── 02-postgres-configmap.yaml
│   ├── 03-clientes-seed-configmap.yaml
│   ├── 04-postgres-pvc.yaml
│   ├── 05-postgres-deployment.yaml
│   ├── 06-postgres-service.yaml
│   ├── 07-pgrest-deployment.yaml
│   ├── 08-pgrest-hpa.yaml
│   └── 09-exposition-service.yaml
└── README.md
```

## Visão geral

Este projeto provisiona uma stack Kubernetes enxuta, orientada a produção, com armazenamento persistente, configuração externa, health checks, inicialização automática do banco e escalonamento horizontal.

## Componentes

| Componente | Função |
| --- | --- |
| PostgreSQL 16 | Banco de dados relacional persistente |
| PostgREST 12.2.3 | Interface REST para o banco de dados |
| PersistentVolumeClaim | Armazenamento persistente de 1 GiB |
| ConfigMaps | Configuração da aplicação e seed do banco |
| Secret | Credenciais do banco e chave JWT |
| HPA | Escalonamento do PostgREST de 1 a 3 réplicas |
| GitHub Actions | Validação dos manifests e workflow de deploy |

## Fluxo de requisições

```text
Client
  |
  v
pgrest-service:80
  |
  v
PostgREST:3000
  |
  v
postgres:5432
  |
  v
PostgreSQL + postgres-pvc
```

## Pré-requisitos

| Requisito | Versão ou detalhe |
| --- | --- |
| Kubernetes | 1.29 ou superior |
| kubectl | Configurado para o cluster desejado |
| Metrics Server | Necessário para o HPA |
| curl | Usado nos smoke tests da API |

```bash
kubectl get nodes
kubectl config current-context
```

## Configuração

Crie o Secret a partir do modelo:

```bash
cp k8s/01-postgres-secret.yaml.example k8s/01-postgres-secret.yaml
```

Preencha os valores abaixo em `k8s/01-postgres-secret.yaml`:

```yaml
stringData:
  POSTGRES_DB: "appdb"
  POSTGRES_USER: "postgres"
  POSTGRES_PASSWORD: "SUA_SENHA"
  PGRST_DB_URI: "postgres://postgres:SUA_SENHA@postgres:5432/appdb?sslmode=disable"
  PGRST_JWT_SECRET: "SEU_JWT_SECRET"
```

## Deploy

Aplique todos os recursos:

```bash
kubectl apply -f k8s/
```

> O PostgreSQL executa o seed automaticamente somente quando o diretório de dados é inicializado pela primeira vez.

O seed cria a tabela `clientes` e insere registros de exemplo.

Confira o estado da aplicação:

```bash
kubectl get all -n desafio-k8s
kubectl get pvc -n desafio-k8s
kubectl rollout status deployment/postgres -n desafio-k8s
kubectl rollout status deployment/pgrest -n desafio-k8s
```

## Testar a API

Em um terminal, abra o acesso local:

```bash
kubectl port-forward -n desafio-k8s service/pgrest-service 3000:80
```

Em outro terminal:

```bash
curl http://127.0.0.1:3000/
curl http://127.0.0.1:3000/clientes
```

Criar um cliente:

```bash
curl -X POST http://127.0.0.1:3000/clientes \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  --data '{"nome":"Maria Lima","email":"maria@example.com","ativo":true}'
```

## Persistência

Recrie o Pod do PostgreSQL e consulte o registro novamente:

```bash
kubectl delete pod -l app=postgres -n desafio-k8s
kubectl rollout status deployment/postgres -n desafio-k8s
curl "http://127.0.0.1:3000/clientes?email=eq.maria@example.com"
```

Os dados continuam disponíveis porque são armazenados no `postgres-pvc`.

Para recriar o banco do zero:

```bash
kubectl delete namespace desafio-k8s
kubectl apply -f k8s/
```

## Escalonamento

```bash
kubectl get hpa -n desafio-k8s
kubectl describe hpa pgrest-hpa -n desafio-k8s
kubectl top pods -n desafio-k8s
```

O HPA requer o Metrics Server e utiliza 70% de uso médio de CPU como alvo.

## CI/CD

O workflow de CI é executado em pull requests e pushes para `main` e `develop`. Ele valida os manifests Kubernetes, as tags das imagens, whitespace e credenciais expostas.

O workflow de CD é executado manualmente em **Actions > CD > Run workflow**. Configure estes secrets no environment `production`:

```text
KUBE_CONFIG_DATA
POSTGRES_PASSWORD
PGRST_DB_URI
PGRST_JWT_SECRET
```

O workflow de CD configura o Secret, aplica os recursos, aguarda os rollouts e executa um smoke test em `/clientes`.

## Documentação dos níveis

Cada etapa do desafio possui uma documentação própria em [`docs/`](docs/):

| Nível | Tema | Documentação |
| --- | --- | --- |
| 1 | Namespace | [nivel1-namespace.md](docs/nivel1-namespace.md) |
| 2 | PostgreSQL | [nivel2-postgres.md](docs/nivel2-postgres.md) |
| 3 | Configuração e segredos | [nivel3-configuracao-segredos.md](docs/nivel3-configuracao-segredos.md) |
| 4 | API PostgREST | [nivel4-api-postgrest.md](docs/nivel4-api-postgrest.md) |
| 5 | Exposição e persistência | [nivel5-exposicao-persistencia.md](docs/nivel5-exposicao-persistencia.md) |
| 6 | Health checks e escala | [nivel6-health-checks-escala.md](docs/nivel6-health-checks-escala.md) |
| 7 | Escalonamento automático | [nivel7-escalonamento-automatico.md](docs/nivel7-escalonamento-automatico.md) |

## Limpeza

```bash
kubectl delete namespace desafio-k8s
```
