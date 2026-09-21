# Desafio Kubernetes: PostgreSQL persistente + PostgREST

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29%2B-326ce5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169e1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![PostgREST](https://img.shields.io/badge/PostgREST-API-ff6b6b?logo=postgresql&logoColor=white)](https://postgrest.org/)
[![Status](https://img.shields.io/badge/Status-Demo%20Project-4caf50)](https://github.com/)
[![License](https://img.shields.io/badge/License-MIT-8a2be2)](LICENSE)

Aplicação em Kubernetes com banco PostgreSQL persistente, API REST exposta via PostgREST, configuração externa em Secrets e ConfigMaps, health checks e escalonamento horizontal com HPA.

![Diagrama da arquitetura](./assets/Diagrama.png)

## Índice

- [Visão geral](#visão-geral)
- [Objetivos](#objetivos)
- [Arquitetura](#arquitetura)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Pré-requisitos](#pré-requisitos)
- [CI/CD](#cicd)
- [Configuração do Secret](#configuração-do-secret)
- [Implantação dos manifests](#implantação-dos-manifests)
- [Popular o banco e testar a API](#popular-o-banco-e-testar-a-api)
- [Verificação de persistência](#verificação-de-persistência)
- [Health checks, recursos e escalonamento](#health-checks-recursos-e-escalonamento)
- [Validação automatizada](#validação-automatizada)
- [Limpeza](#limpeza)
- [Critérios de aceitação](#critérios-de-aceitação)
- [Entregáveis](#entregáveis)
- [Observações finais](#observações-finais)

## Visão geral

Este repositório implementa uma solução simples e prática para rodar uma API REST em Kubernetes, usando:

- PostgreSQL com armazenamento persistente via `PersistentVolumeClaim`
- PostgREST conectado ao banco pelo serviço interno `postgres`
- Segredos para credenciais e variáveis sensíveis
- Configuração externa para parâmetros não sensíveis
- `readiness` e `liveness` para garantir disponibilidade
- `HorizontalPodAutoscaler` para escalar a API por CPU

A ideia principal é demonstrar um ambiente Kubernetes funcional, com persistência de dados e infraestrutura organizada em manifests numerados.

## Objetivos

- Isolar os recursos em um namespace próprio: `desafio-k8s`
- Garantir persistência dos dados do PostgreSQL em PVC
- Manter credenciais fora do código em `Secret`
- Configurar parâmetros não sensíveis no `ConfigMap`
- Conectar a API ao banco pelo Service interno `postgres`
- Expor a API com o Service `pgrest-service`
- Validar a persistência após reinicialização do Pod do banco
- Configurar health checks, requests, limits e HPA

## Arquitetura

```text
Cliente externo
      |
      v
pgrest-service:80 (LoadBalancer / port-forward)
      |
      v
PostgREST (Deployment pgrest, 1..3 replicas)
      |
      v
postgres:5432 (Service interno)
      |
      v
PostgreSQL (Deployment postgres) --> PVC postgres-pvc (1Gi)
```

O PostgreSQL não é escalonado horizontalmente neste desafio. O PostgREST é stateless e pode ter múltiplas replicas, enquanto o banco mantém o estado em um PVC.

## Estrutura do repositório

```text
.
├── k8s/
│   ├── 00-namespace.yaml              # Namespace do projeto
│   ├── 01-postgres-secret.yaml.example # Template do Secret (não versionado)
│   ├── 02-postgres-configmap.yaml     # Configuração não sensível
│   ├── 03-postgres-pvc.yaml           # Armazenamento persistente
│   ├── 04-postgres-deployment.yaml    # PostgreSQL + PVC + probes
│   ├── 05-postgres-service.yaml       # Service interno postgres:5432
│   ├── 06-pgrest-deployment.yaml     # PostgREST + probes
│   ├── 07-pgrest-hpa.yaml            # Escalonamento horizontal
│   ├── 08-exposition-service.yaml    # Service externo da API
│   ├── 03-clientes-seed-configmap.yaml # Seed executado na primeira inicialização
│   └── 09-clientes-seed.sql          # Seed fonte da tabela clientes
├── validate_k8s.py                   # Validação automatizada
├── passo a passo teste.md            # Roteiro complementar
├── Diagrama.png                      # Arquitetura visual
├── .gitignore                        # Ignora secrets locais e arquivos sensíveis
├── README.md                         # Documentação do projeto
└── .github/workflows/                # Pipelines de CI e CD
```

## Pré-requisitos

Antes de iniciar, confirme que você possui:

- Kubernetes local ou remoto: Docker Desktop, kind, minikube, k3d, etc.
- `kubectl` configurado para o cluster correto
- Python 3 para executar `validate_k8s.py`
- Metrics Server instalado para que o HPA calcule consumo de CPU
- `curl` disponível para testes HTTP

Verifique o acesso ao cluster:

```bash
kubectl version
kubectl get nodes
```

> Os exemplos abaixo usam uma shell de terminal compatível com POSIX, como Bash, Zsh, Git Bash ou WSL. Em Windows, também é possível executá-los pelo terminal integrado do VS Code usando Git Bash ou WSL.

## CI/CD

O projeto possui dois workflows em `.github/workflows/`:

- `ci.yml`: executa em pushes para `main`/`master` e em pull requests. Valida o formato do diff, procura segredos expostos com Gitleaks, rejeita imagens sem tag fixa, valida os schemas Kubernetes com Kubeconform e impede que um Secret real seja versionado.
- `cd.yml`: executa somente por `workflow_dispatch`. Faz o deploy no environment `production`, usa permissões mínimas, configura o acesso ao cluster com kubeconfig temporário e cria o Secret Kubernetes a partir de secrets do GitHub, sem armazená-lo no repositório.

### Configuração do CD no GitHub

Crie um environment chamado `production` em **Settings > Environments** e, se necessário, configure aprovação obrigatória antes do deploy. Adicione estes secrets no environment:

- `KUBE_CONFIG_DATA`: conteúdo do kubeconfig codificado em Base64.
- `POSTGRES_PASSWORD`: senha do PostgreSQL.
- `PGRST_DB_URI`: URI completa para conexão do PostgREST, usando a mesma senha de `POSTGRES_PASSWORD`.
- `PGRST_JWT_SECRET`: chave JWT do PostgREST.

Para gerar o valor de `KUBE_CONFIG_DATA` no terminal:

```bash
base64 -w 0 ~/.kube/config
```

Em macOS, use `base64 < ~/.kube/config | tr -d '\\n'`. No PowerShell, use `[Convert]::ToBase64String([IO.File]::ReadAllBytes("$HOME/.kube/config"))`. Nunca coloque esses valores em arquivos versionados ou em logs. Inicie o CD pela aba **Actions**, selecionando `CD` e executando **Run workflow**. O workflow aplica os manifests, aguarda os Deployments ficarem prontos, executa o seed e valida `GET /clientes`; ele não aplica o arquivo `.example` do Secret.

## Configuração do Secret

O arquivo real do Secret não deve ser versionado. O modelo `01-postgres-secret.yaml.example` deve ser copiado para um arquivo local e preenchido com valores reais.

### Criar o Secret local

No terminal, copie o modelo para o arquivo local:

```bash
cp k8s/01-postgres-secret.yaml.example k8s/01-postgres-secret.yaml
```

Edite `k8s/01-postgres-secret.yaml` e preencha pelo menos:

- `POSTGRES_PASSWORD`
- `PGRST_DB_URI` com a senha correta e mantendo o host `postgres`, porta `5432` e banco `appdb`
- `PGRST_JWT_SECRET` em um ambiente real

Exemplo:

```yaml
stringData:
  POSTGRES_DB: "appdb"
  POSTGRES_USER: "postgres"
      POSTGRES_PASSWORD: "CHANGE_ME_POSTGRES_PASSWORD"
      PGRST_DB_URI: "postgres://postgres:CHANGE_ME_POSTGRES_PASSWORD@postgres:5432/appdb?sslmode=disable"
      PGRST_JWT_SECRET: "CHANGE_ME_JWT_SECRET"
```

Nunca publique esse arquivo ou exponha os valores em issues, screenshots ou documentos públicos.

## Implantação dos manifests

A ordem dos manifests é importante. Aplicando em sequência:

```bash
kubectl apply -f k8s/
```

Monitore a criação dos recursos:

```bash
kubectl rollout status deployment/postgres -n desafio-k8s
kubectl rollout status deployment/pgrest -n desafio-k8s
kubectl get pods -n desafio-k8s
kubectl get pvc -n desafio-k8s
```

Resultado esperado:

- `postgres` em `1/1 Running`
- `pgrest` em `1/1 Running`
- `postgres-pvc` em `Bound`

## Popular o banco e testar a API

O seed é executado automaticamente pelo PostgreSQL na primeira inicialização, porque o arquivo é montado em `/docker-entrypoint-initdb.d/`. Portanto, a aplicação pode ser criada com:

```bash
kubectl apply -f k8s/
```

Esse script cria a tabela `clientes`, insere registros de exemplo e dispara `NOTIFY pgrst, 'reload schema'` para atualizar o cache do PostgREST. Scripts de inicialização do PostgreSQL só são executados quando o diretório de dados está vazio; com um PVC já existente, eles não são executados novamente.

Em um segundo terminal, mantenha o port-forward aberto na porta `3000`:

```bash
kubectl port-forward service/pgrest-service 3000:80 -n desafio-k8s
```

Teste a API em outro terminal ou navegador:

```bash
curl http://127.0.0.1:3000/
curl http://127.0.0.1:3000/clientes
```

Acesse também no navegador:

```text
http://localhost:3000/clientes
```

> Se um Pod do PostgREST for recriado, o port-forward antigo pode falhar temporariamente com `failed to find sandbox`. Isso é esperado; aguarde o novo Pod ficar pronto e reinicie o port-forward.

## Verificação de persistência

Crie um novo registro por meio da API:

```bash
printf '%s\n' '{"nome":"Persistencia terminal","email":"persistencia-terminal@teste.com","ativo":true}' > payload.json
curl -X POST http://127.0.0.1:3000/clientes -H "Content-Type: application/json" -H "Prefer: return=representation" --data-binary "@payload.json"
curl "http://127.0.0.1:3000/clientes?email=eq.persistencia-terminal@teste.com"
```

Reinicie o Pod do banco:

```bash
kubectl delete pod -l app=postgres -n desafio-k8s
kubectl rollout status deployment/postgres -n desafio-k8s
kubectl get pods -l app=postgres -n desafio-k8s
```

Depois que o novo Pod estiver em `Running`, execute novamente:

```bash
curl "http://127.0.0.1:3000/clientes?email=eq.persistencia-terminal@teste.com"
rm payload.json
```

O mesmo registro deve continuar disponível após a recriação do Pod. Durante a troca, a API pode responder temporariamente com `57P01` ou `503` enquanto o PostgREST reconecta ao banco.

## Health checks, recursos e escalonamento

Confira os detalhes do Deployment da API:

```bash
kubectl describe deployment pgrest -n desafio-k8s
```

O Deployment da API possui:

- `readiness` e `liveness` HTTP na porta `3000`
- requests de `100m` CPU e `128Mi` de memória
- limits de `500m` CPU e `512Mi` de memória

Observe o HPA e as métricas:

```bash
kubectl get hpa pgrest-hpa -n desafio-k8s
kubectl describe hpa pgrest-hpa -n desafio-k8s
kubectl top pods -n desafio-k8s
kubectl top nodes
```

O HPA escala `pgrest` de 1 a 3 replicas com alvo de 70% de CPU. Se aparecer `cpu: <unknown>/70%`, o Metrics Server ainda não está disponível ou o Pod ainda não está pronto.

Para gerar carga de teste:

```bash
kubectl run load-generator-1 -n desafio-k8s --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://pgrest-service/clientes > /dev/null; done"
```

Em um terminal separado, acompanhe o HPA:

```bash
kubectl get hpa pgrest-hpa -n desafio-k8s -w
```

Em outro terminal, acompanhe as replicas:

```bash
kubectl get pods -l app=pgrest -n desafio-k8s -w
```

Quando terminar, remova a carga:

```bash
kubectl delete pod load-generator-1 -n desafio-k8s
```

## Limpeza

Para remover todo o ambiente:

```bash
kubectl delete namespace desafio-k8s
```

## Critérios de aceitação

- [x] Namespace próprio com recursos isolados
- [x] PostgreSQL em execução com `PersistentVolumeClaim`
- [x] Credenciais fora dos Deployments e protegidas por `Secret`
- [x] Configuração não sensível em `ConfigMap`
- [x] PostgREST conectado pelo Service interno `postgres`
- [x] API exposta pelo Service `pgrest-service`
- [x] Dados inseridos pela API sobrevivem à recriação do Pod do banco
- [x] `readiness` e `liveness` configurados na API
- [x] Requests e limits definidos
- [x] HPA baseado em CPU configurado de 1 a 3 replicas
- [x] Manifests organizados e numerados
- [x] Procedimento de aplicação e teste documentado
