# ArgoCD com Ingress Traefik

![argocd01](https://user-images.githubusercontent.com/52961166/141650959-3050af34-3b36-4b85-b921-c080c4f7264f.png)

# Instalando ArgoCD

Argo CD é uma ferramenta declarativa de entrega contínua para Kubernetes baseada na abordagem GitOps.

| Definições, configurações e ambientes de aplicativos devem ser declarativos e com controle de versão.

| A implantação de aplicativos e o gerenciamento do ciclo de vida devem ser automatizados, auditáveis e fáceis de entender.

Baixe o yaml de instalação do ArgoCD Latest:
```sh
wget -c https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Queremos que ArgoCD esteja disponível em /argocd, então precisamos alterar o caminho adicionando raiz do aplicativo com a opção --rootpath no comando do recipiente do servidor argocd e as linhas --staticassets /shared/app e também adicionar o --insecure opção: 
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/part-of: argocd
  name: argocd-server
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: argocd-server
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: argocd-server
              topologyKey: kubernetes.io/hostname
            weight: 100
          - podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/part-of: argocd
              topologyKey: kubernetes.io/hostname
            weight: 5
      containers:
      - command:
        - argocd-server
        - --staticassets
        - /shared/app
        # Add insecure and argocd as rootpath
        - --insecure
        - --rootpath
        - /argocd
        env:
        - name: ARGOCD_SERVER_INSECURE
          valueFrom:
            configMapKeyRef:
              key: server.insecure
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_SERVER_BASEHREF
```		
Uma vez feito isso, crie o namespace argocd e instale o ArgoCD com o script modificado 
```sh
$ kubectl create namespace argocd
$ kubectl apply -f install.yaml -n argocd
```
Crie uma entrada para redirecionar /argocd para o serviço principal argocd: 		
```sh
nano argocd-ingress.yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.certresolver: default
  generation: 2
  labels:
    app: argocd
  managedFields:
  - apiVersion: extensions/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubernetes.io/ingress.class: {}
          f:traefik.ingress.kubernetes.io/router.tls: {}
          f:traefik.ingress.kubernetes.io/router.tls.certresolver: {}
        f:labels:
          .: {}
          f:app: {}
      f:spec:
        f:tls: {}
    manager: kubectl-client-side-apply
  - apiVersion: extensions/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          f:field.cattle.io/ingressState: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
      f:spec:
        f:rules: {}
    manager: rancher
  name: argocd-ingress
  namespace: argocd
spec:
  rules:
  - host: argocd.domnio.com.br
    http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: 80
        path: /argocd
        pathType: ImplementationSpecific
  tls:
  - secretName: traefik-cert
status:
  loadBalancer: {}
```  
Execute o script para criar a entrada de ingress
```sh
$ kubectl apply -f argocd-ingress.yaml -n argocd
```  
Abra um navegador em https://dominio.com.br/argocd. 

Observe que a instalação pode levar algum tempo para ser concluída. 

![argocd](https://user-images.githubusercontent.com/52961166/141650496-d983f707-2b1e-4ca9-978d-f8a23fd562b5.png)

Por padrão, ArgoCD usa o nome do pod do servidor como a senha padrão para o usuário administrador, então vamos substituí-lo por mysupersecretpassword (usamos https://bcrypt-generator.com/ para gerar a versão de hash blowfish de "mysupersecretpassword" abaixo
```sh
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2y$12$Kg4H0rLL/RVrWUVhj6ykeO3Ei/YqbGaqp.jAtzzUSJdYWT6LUh/n6",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
 ``` 
Agora use as credenciais admin e mysupersecretpassword como senha e devemos obter uma instância ArgoCD vazia e pronta para uso!

![argocd1](https://user-images.githubusercontent.com/52961166/141650521-5a989556-21a6-468c-98d2-455b656c8007.png)

Você pode querer explorar um pouco a IU, mas como queremos automatizar a maior parte de nossa configuração, é melhor não configurar nada manualmente.   

# Aplicativo de Demonstração

Usaremos um repositório público no GitHub como exemplo aqui, mas ArgoCD é agnóstico quanto à hospedagem dos manifestos do aplicativo, desde que eles sigam o protocolo git.

Nosso aplicativo de demonstração é uma implantação de um servidor da web nginx com algum conteúdo (um ConfigMap que armazena o site e montado como um volume), um serviço e uma entrada para torná-lo acessível no / app: 

![argocd01](https://user-images.githubusercontent.com/52961166/141651728-795081ef-a0f7-4c67-8d60-ddf2a70f756f.png)

O aplicativo de demonstração está hospedado publicamente no github: https://github.com/sokube/argocd-rocks

- website-cm.yaml: um site html simples armazenado como um ConfigMap

- website-deployment.yaml: Observe o ConfigMap montado como um volume no caminho de conteúdo html do site nginx padrão

- website-svc.yaml: Um serviço ClusterIP simples para nosso Ingress

- website-ingress.yaml: Por favor, observe as anotações aqui com uma linha específica para traefik reescrever as solicitações de urls / app com /

Como ArgoCD sincroniza um conteúdo de namespace com os manifestos em um caminho dentro de um repositório git, precisamos criar de antemão um namespace dedicado (dev):
```sh
$ kubectl create namespace dev
```
Queremos que nosso trabalho dentro do ArgoCD seja hospedado em um projeto dedicado chamado argocdrocks-project. Colocamos voluntariamente algumas restrições neste projeto, que se aplicarão a todos os aplicativos que dentro deste projeto: permitir apenas implantações em cluster no namespace dev e apenas para os repositórios github Sokube. 
```sh
$ cat > project.yaml << EOF 
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: argocdrocks-project
  labels:
    app: argocdrocks
spec:
  # Project description
  description: Our ArgoCD Project to deploy our app locally
  # Allow manifests to deploy only from Sokube git repositories 
  sourceRepos:
  - "https://github.com/sokube/*"
  # Only permit to deploy applications in the same cluster
  destinations:
  - namespace: dev
    server: https://kubernetes.default.svc
  # Enables namespace orphaned resource monitoring.
  orphanedResources:
    warn: false
EOF
```
```sh
$ kubectl apply -f project.yaml -n argocd
```
A seguir, criaremos um aplicativo ArgoCD que sincronizará nossos manifestos Kubernetes hospedados na pasta do aplicativo em nosso repositório github branch featurebranch_1 com os recursos associados dentro do namespace dev em nosso cluster local:
```sh
$ cat > application.yaml << EOF 
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  labels:
    app: argocdrocks
  name: argocdrocks-app
spec:
  project: argocdrocks-project
  source:
    repoURL: https://github.com/sokube/argocd-rocks.git
    targetRevision: featurebranch_1
    path: app
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
EOF
```
```sh
$ kubectl apply -f application.yaml -n argocd
```
# Vamos parar um momento na seção syncPolicy da definição do aplicativo ArgoCD:

 - As tentativas de sincronização automatizada falharam 5 vezes com os seguintes atrasos entre as tentativas: 5s, 10s, 20s, 40s, 80s. O número de novas tentativas, atrasos e fatores de multiplicação são configuráveis em uma seção de nova tentativa adicional (não ilustrada aqui).

- prune especifica se os recursos devem ser removidos durante a sincronização automática (falso por padrão).

- selfHeal especifica se a sincronização (parcial) deve ser feita se os recursos no cluster de destino do Kubernetes forem alterados enquanto nenhuma mudança git foi detectada (como uma exclusão manual de uma implantação, por exemplo) 
```sh
$ kubectl apply -f application.yaml -n argocd
```
A sincronização é imediata, e nossos vários recursos são criados dentro do nosso namespace dev:

![argocd02](https://user-images.githubusercontent.com/52961166/141651747-4b090d57-84ab-4b89-a7bf-6947519df343.png)

E nosso aplicativo foi encontrado corretamente em /app no endereço localhost: 

![argocd03](https://user-images.githubusercontent.com/52961166/141651755-6773023d-30e2-42b3-84bb-e0d6e9f475bd.png)

Qualquer mudança subsequente em nosso branch de recursos será escolhida por ArgoCD (que regularmente pesquisa o repositório git) e aplicada dentro de nosso cluster. Vamos modificar website-cm.yaml dentro de nosso branch feature_branch1: 
```sh
...
          <h1>ArgoCD Rocks Really!</h1>
          <p>Version 2</p>
...
```
Nós confirmamos, enviamos e assistimos ArgoCD pegar as mudanças e aplicá-las. Aqui, atualizar o ConfigMap não aciona uma reimplantação, mas como o ConfigMap é montado como um volume, a mudança é selecionada pelo kubelet do Kubernetes após um minuto ou mais. 

![argocd04](https://user-images.githubusercontent.com/52961166/141651765-06097cc3-f15a-43ee-bf49-ce5bc1309c0b.png)

# Conclusão

Por meio deste artigo rápido, pudemos demonstrar e experimentar o uso do ArgoCD usando o Git (GitLab) como uma fonte da verdade para a implantação do aplicativo. 

Em um cenário da vida real, o ArgoCD faria parte de um pipeline de CI/CD corporativo seguro para implantar cargas de trabalho em clusters Kubernetes de produção.

Operadores GitOps, como ArgoCD ou FluxCD, definitivamente ajudam na implementação de cenários de Operações por Pull-Requests. 

Ambos os projetos, respectivamente projetos de incubação e sandbox CNCF, realmente decidiram unir forças para trazer uma experiência GitOps unificada, talvez logo padronizada, para a comunidade Kubernetes ... 
