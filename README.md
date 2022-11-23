# :whale2: Kubernetes Utilizando MySQL and Wordpress

## :ballot_box_with_check: Pré-requisitos

Necessária a intalação do docker e Kubernetes

## :whale: Instalação do Docker

para a instalação seguimos o link: 

## Instalação do kubernetes

Para usar o kubernetes iremos utilizar o Linux ubuntu, e os seguintes comandos:

**Dessa maneira é feito o update dos pacotes e se instalam algumas dependencias:**

`$ sudo apt-get update`

`$ sudo apt-get install -y ca-certificates curl`

`$ sudo apt-get install -y apt-transport-https`

**Dessa maneira é feito a instalação do kubernetes:**

`$ sudo apt-get install -y kubectl`

## Agora podemos começar
Para iniciar o projeto se cria um diretório com o comando:

`$ mkdir nomedodiretorio`

Entramos no diretório criado:

`$ cd nomedodiretorio`

Se cria o namespace:

`$ kubectl create namespace <nomeaqui>`

### Criação do Secret

Iremos criar um arquivo para armazenar nossas senhas e manter a segurança do nosso ambiente:

>secrets.yaml

Com o seguinte conteúdo:
```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-password
type: Opaque
data:
  password: suasenhaaqui
```
Cada uma dessas linhas tem as seguintes funções:

- apiVersion:v1 -> Estabelecemos a versão da api.
- kind: Secret -> Definimos o tipo de objeto a ser criado.
- metadata: -> Definimos informações do objeto criado.
- name: mysql-password -> É definido o nome do secret.
- data: -> Definimos os dados do arquivo.
- password: suasenhaaqui -> Senha do nosso.

## Trabalhando com MySQL

### Criamos o arquivo de serviço do mysql

com a seguinte configuração:
```
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3308
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
  ```

Cada uma dessas linhas tem as seguintes funções:
- apiVersion:v1 -> Estabelecemos a versão da api.
- kind: Service -> Definimos o tipo de objeto a ser criado.
- metadata: -> Definimos informações do objeto criado.
- name: mysql -> É definido o nome do serviço.
- labels: -> Definimos valores chave para o serviço
- app: -> Definimos o aplicativo executado.
- spec: -> Definimos especificações da rede do serviço.
- ports: -> Especificamos a porta do serviço: - port: 3308.

### Criamos o PersistentVolumeClaim do MySQL
com a seguinte configuração:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-persistent-storage-lab
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
```

### Criamos o arquivo Deployment do MySql
com a seguinte configuração:
```
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-password	# the one generated before in secret.yml
              key: password
        - name: MYSQL_DATABASE
          value: wp-database
        ports:
        - containerPort: 3308
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage-lab  # which data will be stored
          mountPath: "/var/lib/mysql"
      volumes:
      - name: mysql-persistent-storage-lab   # PVC
        persistentVolumeClaim:
          claimName: mysql-persistent-storage-lab
```

## Iniciando o Wordpress

### Criamos o serviço do worpdress
com a seguinte configuração:
```
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
```

### Criamos o PersistentVolumeClaim do worpdress
com a seguinte configuração:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name:  wordpress-persistent-storage-lab
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
```
### Criamos o deployment do worpdress!

com a seguinte configuração:
```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql:3306
        - name: WORDPRESS_DB_NAME
          value: wp-database
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-password          # generated before in secret.yml
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage-lab
          mountPath: "/var/www/html"          # which data will be stored
      volumes:
      - name: wordpress-persistent-storage-lab
        persistentVolumeClaim:
          claimName: wordpress-persistent-storage-lab
```

E após todas essas configurações realizadas podemos aplicar no nosso Cluster:

`$ kubectl apply -f ./ -R --namespace=seunamespaceaqui`

Lembre-se de estar sempre no diretório raiz da aplicação!

## Realizando o Ingress

Criamos o arquivo com a seguinte configuração:

```
apiVersion: networking.k8s.io/v1
   
kind: Ingress
   
metadata:
   
  name: example-ingress
   
  annotations:
   
    nginx.ingress.kubernetes.io/rewrite-target: /$1
   
spec:
   
  rules:
   
    - host: wordtest.info
   
      http:
   
        paths:
   
          - path: /
   
            pathType: Prefix
   
            backend:
   
              service:
   
                name: wordpress
   
                port:
   
                  number: 8080
```

`$ kubectl apply -f <arquivoingress> --namespace=seunamespaceaqui`

E assim sua aplicação de Wordpress com Mysql estará rodando!

## Utilizando a aplicação

com o comando:
`$ kubectl get ingresss`

podemos ver que na coluna "Hosts" temos o link que iremos usar no navegador para acessar a página
