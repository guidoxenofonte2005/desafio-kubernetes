# Desafio: Fundamentos de Kubernetes na Prática

Projeto do CloudOps Bootcamp (módulo S7 — Kubernetes) que implanta, do zero, uma stack completa em um cluster Kubernetes local: uma API REST (PostgREST) integrada a um banco PostgreSQL, com persistência de dados, configuração externalizada, health checks e escalonamento automático — tudo declarado em manifests YAML versionados.

As respostas às perguntas de reflexão de cada etapa, junto com as evidências (prints) de execução, estão em **[answers.md](./answers.md)**.

## Objetivo do projeto

Simular o trabalho de um DevOps Engineer levando uma stack que hoje roda "na mão" para dentro de um cluster Kubernetes, aplicando de forma prática os principais conceitos de orquestração de contêineres: Namespaces, Pods, Deployments, Services, ConfigMaps, Secrets, volumes persistentes, probes de saúde e escalonamento (manual e automático). O critério central do desafio é a API conseguir encontrar o banco de dados dentro do cluster através do nome do Service (DNS interno), e os dados sobreviverem à destruição e recriação do Pod do banco.

## Arquitetura

```
Você (curl / navegador)
        │  HTTP
        ▼
  Service (api-service)
        │
        ▼
  Deployment da API (PostgREST)
        │  SQL, via nome do Service "database"
        ▼
  Service (database)
        │
        ▼
  Deployment do PostgreSQL + PVC
```

A API nunca se conecta ao banco por IP de Pod — ela usa o nome do Service (`database`), que o DNS interno do cluster resolve para o Pod ativo no momento, mesmo que ele seja recriado e mude de IP.

## Ferramentas utilizadas

| Ferramenta | Propósito |
|---|---|
| **Minikube** | Cluster Kubernetes local usado para todo o desafio — provê o nó, o `kubectl` já configurado, e os addons necessários (como o `metrics-server`, usado no HPA). |
| **kubectl** | CLI de interação com o cluster — aplicação de manifests, inspeção de recursos, execução de comandos dentro dos Pods, e acompanhamento de eventos/logs. |
| **postgres:16** | Imagem oficial do PostgreSQL — o banco de dados relacional da stack. |
| **postgrest/postgrest** | Imagem que expõe automaticamente uma API REST completa sobre qualquer tabela do PostgreSQL, sem necessidade de escrever código de backend. Configurada inteiramente por variáveis de ambiente. |
| **metrics-server** | Addon do cluster que coleta métricas de uso de CPU/memória dos Pods — pré-requisito para o `kubectl top` e para o Horizontal Pod Autoscaler funcionarem. |
| **Apache Bench (ab)** | Ferramenta de geração de carga HTTP, usada para testar o Horizontal Pod Autoscaler da API. |
| **Git** | Versionamento dos manifests e do histórico de evolução do projeto (organizado em branches por etapa: `feature/step_two`, `feature/step_three`, etc.). |

## Estrutura do repositório

```
desafio-kubernetes/
├── manifests/          # todos os manifests YAML, numerados na ordem de aplicação
├── images/              # evidências (prints) organizadas por etapa, referenciadas em answers.md
├── README.md            # este arquivo
└── answers.md            # respostas às perguntas de reflexão + evidências de cada etapa
```

### Propósito de cada manifest

Os arquivos são numerados propositalmente — `kubectl apply -f manifests/` processa os arquivos em ordem alfabética, e vários recursos dependem de outros já existirem (o namespace precisa existir antes de tudo, o Secret precisa existir antes do Deployment que o referencia, etc.). Numerar os arquivos garante que tudo suba corretamente em uma única execução, mesmo em um ambiente totalmente limpo.

| Arquivo | Recurso | Propósito |
|---|---|---|
| `00-namespace.yaml` | Namespace | Cria o namespace `desafio`, isolando todos os recursos do projeto do restante do cluster. |
| `01-persistent-volume.yaml` | PersistentVolume | Reserva um espaço de armazenamento (`hostPath`, 10Gi) no cluster, independente do ciclo de vida de qualquer Pod. |
| `02-volume-claim.yaml` | PersistentVolumeClaim | Solicita e vincula (`Bound`) o armazenamento do PV acima para uso pelo Deployment do banco. |
| `03-postgres-secret.yaml` | Secret | Guarda as credenciais do PostgreSQL (`POSTGRES_USER`, `POSTGRES_PASSWORD`) e a string de conexão completa (`PGRST_DB_URI`) usada pela API — nada disso fica hardcoded nos Deployments. |
| `04-postgres-configmap.yaml` | ConfigMap | Guarda configuração não sensível compartilhada entre banco e API: nome do banco (`POSTGRES_DB`), diretório de dados (`PGDATA`), schema (`PGRST_DB_SCHEMA`) e role anônima do PostgREST (`PGRST_DB_ANON_ROLE`). |
| `05-database-deployment.yaml` | Deployment | Sobe o container `postgres:16`, injeta credenciais do Secret e configuração do ConfigMap, e monta o volume persistente no diretório de dados do Postgres. |
| `06-database-service.yaml` | Service (ClusterIP) | Expõe o banco internamente no cluster sob o nome `database`, permitindo que a API o encontre por DNS em vez de IP fixo. |
| `07-api-deployment.yaml` | Deployment | Sobe o container `postgrest/postgrest`, conectado ao banco via `PGRST_DB_URI` (lida do Secret), com `requests`/`limits` de CPU e memória e probes de `liveness`/`readiness` configuradas. |
| `08-api-service.yaml` | Service (ClusterIP) | Expõe a API internamente no cluster, servindo de ponto único de acesso e de balanceamento entre réplicas. |
| `09-api-autoscale.yaml` | HorizontalPodAutoscaler | Escala automaticamente as réplicas da API (entre 1 e 10) com base no uso médio de CPU em relação ao que foi definido em `requests`. |

## Como aplicar o projeto

Pré-requisito: cluster local com `kubectl` configurado e ao menos um nó em estado `Ready`.

```bash
# confirmar que o cluster está de pé
kubectl cluster-info
kubectl get nodes

# aplicar todos os manifests, na ordem correta
kubectl apply -f manifests/

# acompanhar a criação de todos os recursos do namespace
kubectl get all,pv,pvc,secrets,configmap,hpa -n desafio
```

Como cada arquivo já declara `namespace: desafio` em seu próprio `metadata`, não é estritamente necessário passar `-n desafio` no `apply` — mas ele é necessário em qualquer comando de leitura/inspeção posterior (`get`, `describe`, `logs`, `exec`), já que o `kubectl` não assume namespace automaticamente.

Para remover tudo:
```bash
kubectl delete namespace desafio
kubectl delete pv pv   # PersistentVolume é cluster-scoped: não é removido junto com o namespace
```

## Como testar a integração e a persistência

**1. Criar uma tabela no banco** (a API só expõe tabelas que já existem no schema `public`):
```bash
kubectl exec -it -n desafio deploy/database-deployment -- \
  psql -U postgres -d desafio -c "CREATE TABLE items(id serial primary key, nome text);"
```

O PostgREST monta o cache de schema apenas na inicialização, então se a API já estava rodando antes dessa tabela existir, qualquer requisição vai retornar `404 PGRST205 (Could not find the table)` até o cache ser atualizado. Recarregue sem precisar reiniciar o Pod:
```bash
kubectl exec -it -n desafio deploy/database-deployment -- \
  psql -U postgres -d desafio -c "NOTIFY pgrst, 'reload schema';"
```
(Alternativa, mais custosa: `kubectl rollout restart deployment/api-deployment -n desafio`.)

**2. Expor a API na máquina local:**
```bash
kubectl port-forward svc/api-service 3000:3000 -n desafio
```

**3. Inserir e ler um dado através da API** (em outro terminal):
```bash
curl -X POST http://localhost:3000/items -H "Content-Type: application/json" -d '{"nome":"teste"}'
curl http://localhost:3000/items
```

**4. Provar a persistência** — deletar o Pod do banco e confirmar que o dado sobrevive:
```bash
kubectl delete pod -n desafio -l app=database
kubectl wait --for=condition=Ready pod -l app=database -n desafio --timeout=60s
curl http://localhost:3000/items
```
Se o registro inserido no passo 3 ainda aparecer, a persistência via PVC está funcionando corretamente.

## Como visualizar e monitorar os recursos

```bash
# visão geral de tudo que está rodando no namespace
kubectl get all -n desafio

# estado do armazenamento persistente
kubectl get pv,pvc -n desafio

# detalhes e eventos de um recurso específico (essencial para debugging)
kubectl describe pod <nome-do-pod> -n desafio
kubectl describe hpa api-autoscale -n desafio

# logs de um container
kubectl logs -n desafio deploy/database-deployment
kubectl logs -n desafio deploy/database-deployment --previous   # logs do container anterior, útil em CrashLoopBackOff

# uso de CPU/memória em tempo real (depende do metrics-server)
kubectl top pods -n desafio

# acompanhar mudanças ao vivo (Pods sendo criados/destruídos, HPA escalando)
kubectl get pods -n desafio -w
kubectl get hpa -n desafio -w
```

## Escalonamento automático (HPA)

O Horizontal Pod Autoscaler foi configurado apenas para a API (não para o banco — ver a justificativa em [answers.md](./answers.md), etapa 6), escalando entre 1 e 10 réplicas com base em 50% de utilização média de CPU.

Para reproduzir o teste de carga:
```bash
kubectl port-forward svc/api-service 3000:3000 -n desafio

# em outro terminal
ab -n 100000 -c 50 http://localhost:3000/items

# acompanhar o efeito no HPA
kubectl get hpa -n desafio -w
```

Vale notar que o HPA tem um período de estabilização padrão de 5 minutos para reduzir réplicas após a carga cair — isso evita oscilação constante do número de Pods, mas significa que o scale-down não é imediato.

## Principais dificuldades encontradas

Ao longo do desenvolvimento, alguns problemas se repetiram e vale documentá-los como parte do aprendizado prático do desafio:

- **`kubectl create` vs. `kubectl apply`.** Usar `create` repetidamente falha com `AlreadyExists` assim que um recurso já existe, quebrando a reaplicação dos manifests. A solução foi padronizar o uso de `apply` em todo o fluxo, que cria ou atualiza sem travar.

- **PersistentVolume preso em `Released`.** Como a `reclaimPolicy` do PV é `Retain`, apagar o PVC não libera o PV automaticamente para um novo vínculo — ele fica em `Released`, recusando qualquer PVC novo até ser deletado e recriado manualmente. Isso gerou Pods presos em `Pending` diversas vezes, sempre que o namespace era recriado do zero sem também recriar o PV.

- **Ordem de aplicação dos manifests.** Ao rodar `kubectl apply -f .` em uma pasta sem prefixos numéricos, o `kubectl` processa os arquivos em ordem alfabética — não por dependência. Isso fazia com que Deployments fossem aplicados antes do Namespace existir, gerando erro na primeira execução. A solução foi nomear os arquivos com prefixo numérico (`00-`, `01-`, etc.), refletindo a ordem real de dependência.

- **`initdb: directory ... exists but is not empty`.** Montar o volume persistente diretamente na raiz de `/var/lib/postgresql/data` conflita com arquivos como `lost+found`, criados automaticamente por alguns filesystems, ou com dados remanescentes de inicializações anteriores. A solução foi apontar `PGDATA` para um subdiretório dentro do volume montado.

- **PVC imutável após o bind.** Alterar o valor de `storage:` em um PVC já vinculado (`Bound`) gera erro, pois volumes provisionados estaticamente (como neste projeto, via `hostPath`) não suportam redimensionamento — isso só é possível com provisionamento dinâmico. A correção sempre exigiu deletar e recriar o PVC, nunca aplicar por cima.

- **`PGRST302: Anonymous access is disabled`.** O PostgREST bloqueia toda requisição sem autenticação até que `PGRST_DB_ANON_ROLE` seja explicitamente definida, indicando qual role do PostgreSQL usar para acesso anônimo.

- **Inconsistência de nomes entre Secret e ConfigMap.** Boa parte dos erros de conexão (banco "does not exist", falha na `PGRST_DB_URI`) teve origem em nomes de banco de dados divergentes entre os arquivos — reforçando a importância de manter um único nome de referência (`desafio`) em todo o projeto.

- **Delay do HPA no scale-down.** O `stabilizationWindowSeconds` padrão de 5 minutos fazia parecer que o Horizontal Pod Autoscaler estava "travado" em um número alto de réplicas mesmo após a carga cair — comportamento esperado, não um bug, mas que exigiu investigação via `kubectl describe hpa` para confirmar.

## Fontes e materiais consultados
- [Documentação Oficial do Kubernetes](kubernetes.io/docs)
- [Artigo sobre Postgres no Kubernetes K8s](https://medium.com/@howdyservices9/postgres-on-kubernetes-k8s-bf555289831a)
- [Artigo sobre Deploy do PostgreSQL com Persistent Storage](https://md-imran-sheikh.medium.com/deploying-postgresql-on-kubernetes-with-persistent-storage-step-by-step-guide-8e5287d04f3e)
- [Artigo sobre Postgres no Kubernetes com Deploy, Escalonamento e Gerenciamento](https://www.groundcover.com/blog/postgres-in-kubernetes-how-to-deploy-scale-and-manage)
- [Dúvida no StackOverflow sobre PVC em Pods via Deployment](https://stackoverflow.com/questions/60533371/how-to-mount-a-persistent-volume-on-a-deployment-pod-using-persistentvolumeclaim)
- [Dúvida no StackOverflow sobre uso de Secrets e ConfigMaps](https://stackoverflow.com/questions/36912372/when-to-use-secrets-as-opposed-to-configmaps-in-kubernetes)
- [Dúvida no StackOverflow sobre uso de EmptyDir e PV/PVC](https://stackoverflow.com/questions/79575214/why-use-kubernetes-emptydir-volumes-instead-of-container-filesystem)
- Inteligências Artificiais:
  - Claude (tirar dúvidas sobre tópicos específicos, verificação de respostas/clareza de código, sugestões de correção de erros e geração do README)
  - Gemini (respostas automáticas de buscas do Google sobre os assuntos do desafio)