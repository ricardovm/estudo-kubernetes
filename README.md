# Kubernetes Alura

### Lembrete

- Adicionar poder computacional na mesma máquina: escalabilidade vertical
- Adicionar poder computacional com mais máquinas: escalabilidade horizontal

## Sobre

Gerencia cluster com uma ou mais máquinas (orquestrador de containers)

Além de orquestrador (resources)

## Kubernetes

Tipos de máquinas:

- máquinas master - coordenam e gerenciam o cluster, mantes estado, receber comandos (control plane)

  - api,
  - c-m,
  - sched,
  - etcd - chave/valor

- nodes - execução dos pods

  - kubnlet - execução dos pods
  - k-proxy - comunicação entre nodes


## kubectl

Usa a API (criar, ler, atualizar, remover recursos) declarativo ou imperativo

```bash
kubectl get nodes
```

## Execução local

No Windows: configurar no Docker Desktop

No Linux:

- Baixar o kubectl
- Baixar o minikube
- Rodar com driver virtualbox (necessário rodar sempre que reiniciar a máquina)

## Pod - 1 ou mais containers

É o pod que ganha um IP, não exatamente o container. Num pod com dois containers, eles não podem repetir portas. Acessar o IP do pod acessa diretamente as portas dos containers

Se todos os containers de um pod falham, o pod é destruído e o k8s pode substituí-lo (talvez com outro IP). Mas tendo um container saudável, o pod continua vivo

Comaprtilham volumes, IPC e rede. Containers de um mesmo pod podem acessar um ao outro via localhost

Um pod pode conversar com outro

### Para iniciar um pod:

```bash
kubecrl run nome-do-pod --image=nome-da-imagem:latest
kubectl get pods [--watch]
kubectl describe pod nginx-pod # (vê IP, labels, processo de criação do pod, etc.)
kubectl edit pod nome-do-pod
```

### Criando/manipular e gerenciar pods (e outros recursos) de forma declarativa

```bash
kubectl apply -f arquivo.yaml
# (o mesmo comando é usado para aplicar alterações do pod, como versão de imagem)

kubectl delete -f arquivo.yaml    # para interromper o pod
```

### Acessando o pod (por dentro)

```bash
kubectl exec -it nome-do-pod [--container nome-do-container] -- bash
```

Nota: o IP exibido em `describe` é o IP dentro do cluster, mas até agora não fizemos nenhum mapeamento desse IP com o container interno nem qualquer exposição para o munfo externo

```bash
kubectl get pods -o wide 
# (formata a saída em wide, e nesse caso vai mostrar o IP do pod)
```

## Services

Expõe aplicações, provê IP fixo, DNS e load-balancing, e podem ser dos tipos:

- ClusterIP
- NodePort
- LoadBanancer

### ClusterIP

- Service não é via de mão dupla: se um pod tem svc, não significa que ele vai se comunicar de maneira estável com outros pods... apenas os outros vão poder se comunicar com ele via svc
- O ClusterIP é algo interno do cluster e não oferece acesso externo
- A ligação entre svc e pod se dá por Label (label não precisa ser o nome do pod, mas o par chave-valor do label deve ser igual no `metadata` e `selector`)
- definir portas de escuta e despacho (`targetPort`) (ou número simples quando são o mesmo valor)

```bash
kubectl get svc # (para exibir services)
```

Nota: o IP é dentro do cluster, ok? Para testar, `kubectl exec` em outro pod

### NodePort

- Um service, mas para o mundo externo e também funciuona como ClusterIP
- Ao consultar com `kubectl get svc`, o IP vai estar na coluna `cluster-ip`... ainda é o interno. Acesso externo é feito pelo node 

```bash
kubectl get nodes -o wide
```

Nota: pegadinha do Windows... ele mapeia o docker-desktop para localhost. Assim é possível acessar o serviço usando `localhost:aquela-porta-estranha-do-get-svc`. No Linux, usar a coluna `internal-ip` mais a porta estranha (nodePort)

- Para evitar a porta estranha, definir `nodePort` entre 30000-32767

### Load Balancer

- Se integra com o load balancer do cloud provider
- Configuração igual do NodePort, mas com `type: LoadBalancer`

## Config Map - Variáveis ambiente

- Alguns containers (db, por exemplo) precisam de informações por variáveis ambiente. Apesar de poder definir junto as variáveis ambiente, melhor desacoplar a estrutura do pod das configurações.

## ReplicaSet

Aqui pode-se definir um ou mais pods. Quando um pod cai, o RS cria outro. O RS sempre tenta manter o número de réplicas ativas conforme a configuração.

Como os services apontam para os pod por meio dos Labels, a entrada de mais réplicas cria um balanceamento de carga entre os pods.

Dar atenção ao selector -> matchLabels... tem que ser o mesmo definido no template.

## Deployment

Os deployments são ReplicaSets (aparecem na lista de RS), mas têm a função de controlar versão.

```bash
kubectl rollout history deployment nginx-deployment
kubectl apply -f nginx-deployment.yaml –record=true  ## acrescenta um rollout
kubectl annotate deployment nginx-deployment kubernetes.io/change-cause="Definindo imagem com versão latest"  ## para deixar uma descrição legível

kubectl rollout undo deployment nginx-deployment –to-revision=2  ## rollback!
```

Assim, geralmente se usa deployments em vez de RS por conta desse controle de versão

## Volumes

Um volume é atrelado a um pod. Usado para compartilhar arquivos entre containers do pod. Sobrevive à falha de um container, mas é atrelado ao pod. Assim, quando o pod morre, o volume morre (não necessariamente os arquivos).

Para criar um diretório no minikube, pode usar

```bash
minikube ssh
```

Ou com o `type: DirectoryOrCreate`

## PersistentVolume

São volumes na cloud que apontam para discos criados na cloud através do tipo do volume (como um driver... cada tipo pode ter uma forma de apontar. GCP usa `gcePersistentDisk: pdName: nome-do-disco`)

## PersistentVolumeClaim

É uma solicitação para usar os recursos do PV. A ligação é feita através do `accessMode` e tamanho (e `storageClassName`?)

O uso em um pod é parecido com o volume, mas com `persistentVolumeClaim: claimName: nome`.

## StorageClass

Permite criar o PV e discos dinamicamente quando é feito o bind de um pod.

```bash
kubectl get sc
```

A ligação entre o PVC e o SC se dá através do `stotageClassName` (a plataforma já vem com um chamado `standard`). Ao criar o PVC, o PV e o disco são criados. A ligação com o pod é feita normalmente com o PVC.

A remoção do SC também remove o disco.

## StatefulSet

Uma forma de manter o estado do pod com PV e PVC. Cada pod ganha uma identidade e, ao falhar e ser reniciado, ganha a mesma identidade. 

Funciona como um Deployment

Mesmo tendo label, é necessário informar o `serviceName` que vai gerenciar o StatefulSet

Os PV e PVC precisam ser definidos (mas o PV pode ser criado dinamicamente no StorageClass `standard`)

## Liveness e Readiness probes

Lembrar de declarar a porta do container, não a do svc.

## HorizontalPodAutoscaler

O HPA aponta para o recurso que vai gerenciar. A métrica dele também é baseada no percentual do recurso do objeto gerenciado.

Para obter as informações é necessário um servidor de métricas. No Windows, pode baixar de https://github.com/kubernetes-sigs/metrics-server (o YAML em releases). No Linux, utilizamos recursos do minikube com:

```bash
minikube addons enable metrics-server
```

Além do HorizontalPodAutoscaler, o Kubernetes possui um recurso customizado chamado VerticalPodAutoscaler. O VPA remove a necessidade de definir limites e pedidos por recursos do sistema, como CPU e memória. Quando definido, ele define os consumos de maneira automática baseada na utilização em cada um dos nós, além disso, quanto tem disponível ainda de recurso. Mais informações na documentação de cada cloud provider e na documentação do K8S.