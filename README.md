# Configurando cotas de memória e CPU para um namespace no Kubernetes

Esse documento mostra como configurar cotas para memória e CPU que podem ser usadas para Pods rodando em um determinado namespace. Você pode especificar cotas utilizando o objeto **ResourceQuota** do Kubernetes.

## Pré-requisitos:

Para seguir esse documento, você deve possuir os seguintes requisitos:

* Cluster Kubernetes em funcionamento
* Ferramenta de linha de comando **kubectl** precisa está configurada para comunicar com ele.
* Permissão para criar namespaces no cluster
* Cada nó do cluster precisa ter pelo menos 1 GiB de memória