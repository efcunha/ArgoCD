# Instalando ArgoCD com Ingress Traefik

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


