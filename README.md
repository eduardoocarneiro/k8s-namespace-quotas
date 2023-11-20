# Configurando cotas de memória e CPU para um namespace no Kubernetes
Esse documento mostra como configurar cotas para memória e CPU que podem ser usadas para Pods rodando em um determinado namespace. Você pode especificar cotas utilizando o objeto <a href="https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/resource-quota-v1/">ResourceQuota</a> do Kubernetes.

## Pré-requisitos:
Para seguir esse documento, você deve possuir os seguintes requisitos:

* Cluster Kubernetes em funcionamento
* Ferramenta de linha de comando **kubectl** precisa está configurada para comunicar com ele
* Permissão para criar namespaces no cluster
* Cada nó do cluster precisa ter pelo menos 1 GiB de memória

Com isso já podemos iniciar os procedimentos.

## Criando o namespace
Crie um namespace para que os recursos criados neste exercício fiquem isolados do restante do cluster.

```
kubectl create namespace ns-quota-mem-cpu
```

## Criando o ResourceQuota
Agora vamos criar o **ResourceQuota**. Esse recurso vai definir os requests e limits do namespace. Execute o comando abaixo para criar o arquivo de configuração do ResourceQuota:

```
kubectl apply -f yamls/resourcequota.yaml
```

Para visualizar as informações detalhadas do ResourceQuota:

```
kubectl get resourcequota quota-mem-cpu -n ns-quota-mem-cpu --output=yaml
```

O ResourceQuota criado no passo anterior exige esses requisitos no namespace **ns-quota-mem-cpu**:

* Cada Pod no namespace, precisa ter um memory request, memory limit, cpu request, e cpu limit.
* O memory request total para todos os Pods no namespace não pode exceder 1 GiB.
* O memolry limit total para todos os Pods no namespace não pode exceder 2 GiB.
* O CPU request total para todos os Pods no namespace não pode exceder 1 cpu.
* O CPU limit total para todos os Pods no namespace não pode exceder 2 cpu.

> Nota: Altere os valores de CPU e memória da forma que melhor se adeque à sua realidade.

> Para entender melhor o significado de CPU acesse a <a href="https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-cpu">documentação oficial</a>


## Criando o Pod
Terminada a criação do ResourceQuota, vamos agora criar o Pod com os requisitos descritos acima.

Utilizando o arquivo [deployment-1.yaml](yamls/deployment-1.yaml) do diretório yamls, rode o comando abaixo para criar o deployment que iniciará o Pod que precisamos:

```
kubectl apply -f yamls/deployment-1.yaml
```

Verifique se o Pod está rodando:

```
kubectl get pods -n ns-quota-mem-cpu
```

Verifique novamente as informações detalhadas do ResourceQuota:

```
kubectl get resourcequota quota-mem-cpu -n ns-quota-mem-cpu --output=yaml
```

Você perceberá que agora temos o campo **used** como mostra esse trecho da saída do comando executado no passo anterior. Esse campo identifica a quantidade de recursos utilizados no namespace **ns-quota-mem-cpu**. Repare que os recursos utilizados são exatamente os mesmos que o nosso Pod está configurado para consumir.

```
status:
  hard:
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.cpu: "1"
    requests.memory: 1Gi
  used:
    limits.cpu: 800m
    limits.memory: 800Mi
    requests.cpu: 400m
    requests.memory: 600Mi
```

## Tentativa de criar o segundo Pod
Agora vamos criar o segundo Pod com base no arquivo [deployment-2.yaml](yamls/deployment-2.yaml).

Esse segundo deployment, tem o memory request de 700 MiB. Note que a soma do memory request já utilizado pelo primeiro deployment (600 MiB) e esse novo memory request, excede a cota total do namespace **ns-quota-mem-cpu**:

* Memory request deployment 1: 600 MiB
* Memory request deployment 2: 700 MiB
* Total memory request dos dois deployments: 1.3 GiB
* ResourceQuota de memory limit do namespace: 1 GiB

Vamos criar agora o segundo deployment:

```
kubectl apply -f yamls/deployment-2.yaml
```

Ao verificarmos as informações detalhadas dos deployments, perceberemos que o Pod não inicializou corretamente no deployment **quota-mem-cpu-2**.

```
kubectl get deploy -n ns-quota-mem-cpu

NAME              READY   UP-TO-DATE   AVAILABLE   AGE
quota-mem-cpu-1   1/1     1            1           11s
quota-mem-cpu-2   0/1     0            0           2s
```

Para analisarmos o que ocorreu, precisamos olhar os logs do replicaset:

```
kubectl describe replicaset -n ns-quota-mem-cpu
```

Repare que o segundo replicaset, referente ao deployment 2, não inicializou, e o erro é justamente **exceeded quota** como mostrado abaixo.

```
Events:
  Type     Reason        Age                From                   Message
  ----     ------        ----               ----                   -------
  Warning  FailedCreate  73s (x9 over 11m)  replicaset-controller  (combined from similar events): Error creating: pods "quota-mem-cpu-2-67f9cfcd4f-xcrhp" is forbidden: exceeded quota: quota-mem-cpu, requested: requests.memory=700Mi, used: requests.memory=600Mi, limited: requests.memory=1Gi
```

## Conclusão
Como pôde ser visto, você pode usar um ResourceQuota para restringir o memory request total para todos os Pods que rodam em um determinado namespace. Você pode também restringir o total de memory limit, cpu request e cpu limit.

## Clean up
Para limpar toda a configuração realizada, basta apenas deletar o namespace:

```
kubectl delete namespace ns-quota-mem-cpu
```