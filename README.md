# Live Possuidão no YouTube da Linuxtips

## Provas de certificação CKA, CKAD e CKS da Linux Foundation

### Dicas super importantes

    - Todas as perguntas nas provas são práticas. E nas três provas tem questões iguais ou parecidas.

    - Usar killer koda para simulados.

    - Por ser um prova com tempo para finalizar, sempre priorizar comandos diretos (abreviados quando possível) ao invés de usar manifestos. Exemplo: kubectl create ns aovivo é melhor que kubectl create namespace aovivo. Muitos problemas para resolver, precisa estar treinado. 

    - Quando a mais de um container por pod, eles estão mesma camada de rede, o que significa que podem se comunicar por localhost. 

    - Service Account provê uma identidade para o pod em execução, é ele que vai ter ou não permissões ou não para interagir com a API do K8S. 

    - A service account é criada sempre dentro de um namespace. Por padrão já existe uma SA chamada default em cada NS. 

    - Backup do etcd - https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster

    - Saber aplicar, o porque e encontrar na doc. é mais importante que ter os comandos decorados.

    - Treinar encontrar tópicos na doc, principalmente que não sejam usais como kubectl create, kubectl get, kubectl describe.

    - Não gesticular, falar, colocar a mão no rosto ou coisa do tipo.

    - Cybermonday e Linuxtips com desconto


### Exercícios:

1. Criar o namespace "aovivo" com a label cka=true

`kubectl create ns aovivo`
    
    Criar o name space nomeado como "aovivo"

``kubectl edit ns aovivo``

    Inserir a label cka igual a true no name space "aovivo"
    Editar o labels da seção metadata, com aspas (dessa forma):

    metadata:
     creationTimestamp: "2023-12-03T18:15:14Z"
     labels:
      cka: "true"
      kubernetes.io/metadata.name: aovivo

kubectl get ns --show-labels

    Valida a criação do namespace com a LABEL cka=true

2. Criar um deployment "ex2" com 2 containers: a. Imagem Nginx e b. Imagem Redis

kubectl create deployment ex2 --image nginx --dry-run=client -o yaml > ex2.yaml

    Simula a criação (--dry-run=client) de um deployment com as características desejadas
    exportando para um arquivo .yaml (-o yaml > ex2.yaml). Para posterior edição (ver resultado no ex2.yaml)

kubectl apply -f ex2.yaml -n aovivo

    Aplica o manifesto do arquivo (-f ex2.yaml) ex2.yaml no contexto (-n aovivo) do namespace "aovivo" 

kubectl get deployments -n aovivo

    Lista os deployments no contexto do namespace "aovivo"

kubectl get pods -n aovivo -w

    Lista os pods no contexto do namespace "aovivo", vendo o work in progress (-w)

kubectl get deployments.apps -n aovivo

    Valida a criação dos deployments com os pods necessários

3. Testar a comunicação entre esses 2 containers.

kubectl exec -it ex2-774bf99584-wrz68 -c nginx -n aovivo -- bash

    Acessa de forma iterativa um dos containeirs (-c nginx) do pod ex2-774bf99584-wrz68, no contexto do namespace "aovivo" e executando o comando bash. 

        curl -v telnet://0:6379
        *   Trying 0.0.0.0:6379...
        * Connected to 0.0.0.0 (127.0.0.1) port 6379 (#0)
        ping
        +PONG

        Já dentro do container a., no bash, faz um curl para testar conectividade (via telnet) na porta de serviço destino do container b. (redis), e faz um ping que retorna +PONG como resposta válida, confirmando a conexão. 

4. Criar um serviço do tipo NodePort e utilizando a porta 8080.

kubectl expose deployment -n aovivo ex2 --type NodePort --port 8080 --target-port 80

    Cria o serviço do tipo NodePort na porta 8080, definindo a porta do endpoint (alvo/destino) como 80
    
    -- NodePort, comunica-se com o mundo externo usando uma porta TCP entre 30000 e 32767, 
    e fica disponível via IP, como um ponto de acesso único para comunicação diretamente com o POD

 kubectl describe svc -n aovivo ex2

    Descreve o serviço criado, mostrando o IP, endpoint e porta, porta do node etc.

5. Criar uma service account.

kubectl create sa sa-aovivo -n aovivo ou kubectl create serviceaccount sa-aovivo -n aovivo

    Cria a service account sa-aovivo no namespace aovivo

kubectl get sa -n aovivo

    Lista as service account do namespace aovivo

6. Alterar a service account "sa-aovivo" no deployment "ex2".

Pode editar diretamente no arquivo ex2.yaml, de forma a ter um backup e caso precise aplicar novamente já estará pronto. 

    spec:
        serviceAccountName: sa-aovivo
        containers:
        - image: nginx
            name: nginx
            resources: {}

kubectl edit deploy -n aovivo ex2

    Editar diretamente no deployment (na altura da spec de containers)

       spec:
        containers:
        - image: nginx
            name: nginx
            resources: {}
        serviceAccountName: sa-aovivo

kubectl describe deploy -n aovivo ex2

    Valida a service account a nível do deployment

kubectl describe po -n aovivo ex2

    Valida a service account a nível dos pods

7. Criar uma RBAC permitindo listar todos os pods no namespace "aovivo", adicione essa permissão para a SA (service account) criada no exercício 5 e faça o teste utilizando o deployment criado no exercício 2.

kubectl create role -n aovivo listpods --resource pod --verb list

    Cria uma role instanciada no ns "aovivo" chamada listpods, aplicada nos pods (recursos) usando o verbo LISTAR

kubectl create rolebinding -n aovivo listb --role listpods --serviceaccount aovivo:sa-aovivo

    Cria uma rolebinding para associar a role listpods à service account sa-aovivo

kubectl exec -it ex2-5bc96769b-tbhvm -c nginx -n aovivo -- bash

    Acessa de forma iterativa um dos containeirs (-c nginx) do pod, no contexto do namespace "aovivo" e executando o comando bash. 

        env 

            lista as variáveis de ambiente do container

        curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/version

            Realiza uma interação insegura (-k) com a API usando o curl, que retorna com sucesso a versão da API do K8s

        curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/api

            Realiza uma interação insegura (-k) com a API usando o curl, que retorna uma operação proibida, pois não tem permissão 
        
        TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

            cria a variável TOKEN com o conteúdo do token da sa sa-aovivo desse container
        
        curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/api --header "Authorization: Bearer $TOKEN"

            Realiza uma interação insegura (-k) com a API usando o curl autenticando-se com a sa do container, que retorna uma operação de sucesso, confirmando a autenticação.
        
        curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/api/v1/namespaces/aovivo/pods/ --header "Authorization: Bearer $TOKEN"

            Realiza uma interação insegura (-k) com a API usando o curl autenticando-se com a sa do container, que retorna uma operação de sucesso, listando os pods como configurado na role listpods.
        
        curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/api/v1/namespaces/aovivo/services/ --header "Authorization: Bearer $TOKEN"

        Realiza uma interação insegura (-k) com a API usando o curl, listando os services no ns aovivo, que retorna uma operação proibida, pois não tem permissão para listar services, apenas pods. 

        {
            "kind": "Status",
            "apiVersion": "v1",
            "metadata": {},
            "status": "Failure",
            "message": "services is forbidden: User \"system:serviceaccount:aovivo:sa-aovivo\" cannot list resource \"services\" in API group \"\" in the namespace \"aovivo\"",
            "reason": "Forbidden",
            "details": {
                "kind": "services"
            },
            "code": 403
        }




