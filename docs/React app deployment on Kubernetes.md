# React app deployment on Kubernetes with kubectl, kustomize and helm in a multi-environment setup
[https://dev.to/ama/react-app-deployment-on-kubernetes-with-kubectl-kustomize-and-helm-in-a-multi-environment-setup-5b1o]

A maioria das aplicações depende de fatores externos que possuem valores diferentes, dependendo do ambiente em que estão implantado. Usamos principalmente para isso _variaveis e ambiente_. É apresentada uma maneira mais limpa de fazer uma implantação em vários estágios de uma aplicação React em um cluster Kubernetes.

O aplicativo de exemplo exibe os últimos indicadores públicos publicados em www.bookmarks.dev. Dependendo do ambiente para o qual o aplicativo foi criado, ele exibirá o nome do ambiente na barra de navegação e a cor do cabeçalho é diferente.

O código fonte está disponível no [Github](https://github.com/CodepediaOrg/multi-stage-react-app-example)

## TLDR; [^1]
Crie um arquivo _config.js_ no qual você injeta as variáveis ​​de ambiente no objeto `window` (por exemplo, window.REACT_APP_API_URL = 'https: //www.bookmarks.dev/api/public/bookmarks'). Adicione esse arquivo à pasta _public_ da  aplicação React. **Dockerize a apicação react e, no momento do deployment no Kubernetes, substitua o arquivo config.js no contêiner** - isso ser feito no Kubernetes utlizando configMaps, através de comandos nativos do kubectl, kustomize ou helm.

## Pre-requisites
Teste local pode e feito com Docker Destop com Kubernetes ativado, ou com o minikube ou diretamente na nuvem.

## React App Setup
O aplicativo react apresentado neste tutorial é construído com [create-react-ap](https://github.com/facebook/create-react-app)

### A pasta _public_
Você precisa adicionar um _config.js_ na pasta _public_. Isso não será processado pelo webpack [^2]. Em vez disso, ele será copiado para a pasta _build_ inalterado. Para fazer referênciar o arquivo na pasta pública, você precisa usar a variável especial chamada `PUBLIC_URL`:

~~~js
<head>
       .....
       <title>React App</title>
       <script src="%PUBLIC_URL%/config.js"></script>
</head>
~~~

O conteúdo do arquivo _config.js_:

~~~js
    window.REACT_APP_API_URL='https://www.bookmarks.dev/api/public/bookmarks'
    window.REACT_APP_ENVIRONMENT='LOCAL'
    window.REACT_APP_NAVBAR_COLOR='LightBlue'
~~~

> Normalmente, o API_URL apontará para um URL diferente, dependendo do ambiente, mas aqui é o mesmo em geral.

E assim que pode ser definda suas variáveis ​​de ambiente no objeto `windows`. Estas são as propriedades mencionadas acima. Verifique se eles são únicos, portanto, uma boa prática é adicionar o prefixo `REACT_APP_`, conforme sugerido em portanto, uma boa prática é adicionar o prefixo REACT_APP_, conforme sugerido em [Adding Custom Environment Variables.|Adicionando variáveis ​​de ambiente personalizadas.](https://create-react-app.dev/docs/adding-custom-environment-variables/)

> :warning:	WARNING
> Não armazene nenhum segredo (como chaves de API privadas) no seu aplicativo React! As variáveis ​​de ambiente são incorporadas à compilação, o que significa que qualquer pessoa pode visualizá-las, inspecionando os arquivos do aplicativo.

Nesse ponto, você pode executar e criar o aplicativo localmente da maneira que você o conhece:

~~~bash
    npm install 
    npm start
~~~

> É recomendado usar o nvm para executar o NodeJS localmente

e acesse-o em http://localhost:3000

### Por que não usar a abordagem process.env apresentada em Adicionando variáveis ​​de ambiente personalizadas
O tempo de execução de aplicativos da Web estáticos é o navegador, no qual você não tem acesso `process.env`, portanto, os valores dependentes do ambiente devem ser definidos antes disso, ou seja, no momento do build.  

Se você fizer o deploy na sua máquina local, poderá controlar facilmente as variáveis ​​de ambiente - crie o aplicativo para o ambiente que você precisa e faça o deploy. Ferramentas como kustomize e skaffold, fazem com que isso pareça uma brisa no mundo Kubernetes.

Mas se você seguir uma abordagem de _continuous deployment_, normalmente terá várias etapas, que formam o chamado pipeline:
1. commit seu código em um repositório, hospedado em algum lugar como o GitHub;
2. seu sistema de _build_ é notificado;
3. o sistema de build compila o código e executa testes unitário;
4. crie uma imagem e envie-a para um registry, como o Docker Hub.
5. a partir daí você pode fazer o dpeloy da imagem.

A idéia é repetir o mínimo de passos possível para os diferentes ambientes. Com a abordagem apresentada, será apenas a etapa número cinco (deployment/implantação), onde temos configurações específicas do ambiente.


## Containerize a aplicação
Para começar, vamos criar um contêiner de docker para usar no deployment do Kubernetes. Containerizar o aplicativo requer uma imagem base para criar uma instância do contêiner.


### Crie o Dockerfile
O **Dockerfile no diretório raiz do projeto** contém as etapas necessárias para criar a imagem do Docker:

~~~yaml
# build environment
FROM node:12.9.0-alpine as build
WORKDIR /app

ENV PATH /app/node_modules/.bin:$PATH
COPY package.json /app/package.json
RUN npm install --silent
RUN npm config set unsafe-perm true #https://stackoverflow.com/questions/52196518/could-not-get-uid-gid-when-building-node-docker
RUN npm install react-scripts@3.0.1 -g --silent
COPY . /app
RUN npm run build

# production environment
FROM nginx:1.17.3-alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
~~~

Ele usa uma compilação de vários estágios para criar a imagem do docker. Na primeira etapa, você cria o React APP em uma imagem _node alpine_ e, na segunda etapa, você o implanta em uma imagem nginx-alpiine.

### Build the docker image
Para criar a imagem do Docker, execute o seguinte comando no diretório raiz do projeto:
    
`docker build --tag multi-stage-react-app-example:latest . `

Nesse ponto, você pode executar a aplicação no docker usando o seguinte comando:

`docker run -p 3001:80 multi-stage-react-app-example:latest`

Encaminhamos a porta nginx 80 para 3001. Agora você pode acessar o aplicativo em _http://localhost:3001_

> Observe que o ambiente é LOCAL, pois usa o arquivo _config.js_ "original"

### Push para o repositório Docker.
Você também pode enviar a imagem para um repositório docker. Aqui está um exemplo de envio (push) para a organização _codepediaorg_ no dockerhub:

```sh
    docker tag multi-stage-react-app-example codepediaorg/multi-stage-react-app-example:latest
    docker push codepediaorg/multi-stage-react-app-example:latest
```

## Deployment no Kubernetes
Agora você pode pegar um contêiner docker com base na imagem que você criou e implantá-lo no kubernetes.

Para isso, tudo que você precisa fazer é criar o service e deployment do Kubernetes.
**Service**
~~~yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: service
  name: multi-stage-react-app-example
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: multi-stage-react-app-example
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: service
  name: multi-stage-react-app-example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multi-stage-react-app-example
  template:
    metadata:
      labels:
        app.kubernetes.io/component: service
        app: multi-stage-react-app-example
    spec:
      containers:
        - name: multi-stage-react-app-example
          image: multi-stage-react-app-example:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
~~~

## Contexto e namespace Kubernetes
Antes de executar qualquer comando `kubectl apply`, é importante saber em que contexto e namespace ao qual você está aplicando seu comando.

A maneira mais fácil de verificar isso é instalar o kubectx e emitir o `kubectx` para obter o contexto atual e `kubens` para o namespace atual. O namespace padrão é geralmente chamado de `default`. Nesta postagem do blog, operamos no contexto local do `docker-desktop` e no namespace `default`.

Agora que você sabe onde seus objetos kubernetes serão aplicados, você pode adicioná-los a um arquivo, como `deploy-to-kubernetes.yaml` e aplique o seguinte comando:

`kubectl apply -f deploy-to-kubernetes.yaml`

Isso criará o serviço `multi-stage-react-app-example `tipo NodePort.

Você pode verificar sua presença listando todos os serviços

`kubeclt get svc`

ou 

`kubectl get svc | grep multi-stage-react-app-example`


### Port forward
Para acessar o aplicativo dentro do cluster Kubernetes, você pode usar o encaminhamento de porta. O comando para encaminhar o serviço criado antes é: 

`kubectl port-forward svc/multi-stage-react-app-example 3001:80`

> Observe svc antes do nome do serviço

Este comando encaminha a porta local 3001 para a porta 80 do contêiner especificado no arquivo de deployment.

Agora você pode acessar a aplicação dentro do contêiner em http://localhost:3001, que usa o ambiente LOCAL.

## Derrubar objetos Kubernetes criados
Para excluir o serviço e o dpeloy criados, exeute o seguinte comando:

`kubectl delete -f deploy-to-kubernetes.yaml`

## Tornar o deploy da aplicaçãoo ciente do ambiente
Lembre-se do nosso objetivo para o pipeline de entrega contínua: Torne a aplicação "ciente" do ambiente no deployment.

#### Crie um configMap
Começamos criando um configMap.

Criaremos um para o ambiente de `desenvolvimento` a partir do arquivo `environment/dev.properties`

`kubectl create configmap multi-stage-react-app-example-config --from-file=config.js=environment/dev.properties`

Isso cria um configMap, ao qual você pode referenciar pela chave `config.js` e o conteúdo são as variáveis ​​de ambiente.

Você pode verificar isso emitindo o seguinte comando kubectl:

`kubectl get configmaps multi-stage-react-app-example-config -o yaml`

O resultado deve ser algo como o seguinte:

~~~yml
apiVersion: v1
data:
  config.js: |
    window.REACT_APP_API_URL='https://www.bookmarks.dev/api/public/bookmarks'
    window.REACT_APP_ENVIRONMENT='DEV'
    window.REACT_APP_NAVBAR_COLOR='LightGreen'
kind: ConfigMap
metadata:
  creationTimestamp: "2019-08-25T05:20:17Z"
  name: multi-stage-react-app-example-config
  namespace: default
  resourceVersion: "13382"
  selfLink: /api/v1/namespaces/default/configmaps/multi-stage-react-app-example-config
  uid: 06664d35-c6f8-11e9-8287-025000000001Å
~~~

#### Monte o configMap no contêiner
O truque agora é montar o configMap no contêiner por meio de um volume e substituir o arquivo _config.js_ pelos valores do configMap. Mova agora a configuração dos recursos service e deployment para arquivos separados na pasta kubernetes.

O arquivo de deployment:

~~~yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: service
  name: multi-stage-react-app-example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multi-stage-react-app-example
  template:
    metadata:
      labels:
        app.kubernetes.io/component: service
        app: multi-stage-react-app-example
    spec:
      containers:
        - name: multi-stage-react-app-example
          image: multi-stage-react-app-example:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          volumeMounts:
            - name:  multi-stage-react-app-example-config-volume
              mountPath: /usr/share/nginx/html/config.js
              subPath: config.js
              readOnly: true
      volumes:
        - name: multi-stage-react-app-example-config-volume
          configMap:
            name: multi-stage-react-app-example-config
~~~

Na seção `volumes` da especificação, defina um volume com base no configMap que você acabou de criar:

~~~yml
    volumes:
        - name: multi-stage-react-app-example-config-volume
          configMap:
            name: multi-stage-react-app-example-config
~~~

e monte-o no contêiner na pasta de onde o nginx entrega seus arquivos:

~~~yml
spec:
  ...
  template:
  ...
    metadata:
      labels:
        app.kubernetes.io/component: service
        app: multi-stage-react-app-example
    spec:
      containers:
        ...
          volumeMounts:
            - name:  multi-stage-react-app-example-config-volume
              mountPath: /usr/share/nginx/html/config.js
              subPath: config.js
              readOnly: true
~~~

> :pushpin: Nota: você precisa usar o `subpath` para substituir apenas o arquivo config.js, caso contrário, o conteúdo da pasta será substituído por esse arquivo.

#### Deploy no cluster "dev" do kubernetes
Usaremos o mesmo cluster local para testar nossa implantação de desenvolvimento. Você aplica agora o kubectl em todos os arquivos no diretório `kubernetes`:

`kubectl apply -f kubernetes`

Verifique se o arquivo _config.js_ foi substituído conectando-se ao pod:

~~~sh
# first export list the pod holding our application | primeir exportar lista o pod que contém nosso aplicativo
export MY_POD=`kubectl get pods | grep multi-stage-react-app-example | cut -f1 -d ' '`

# connect to shell in alpine image | conectar ao shell na imagem alpina

kubectl exec -it $MY_POD -- /bin/sh 

# display content of the config.js file | exibir o conteúdo do arquivo config.js
less /usr/share/nginx/html/config.js 
~~~

Ele deve conter as variáveis ​​para o ambiente de `desenvolvimento`:

~~~js
    window.REACT_APP_API_URL='https://www.bookmarks.dev/api/public/bookmarks'
    window.REACT_APP_ENVIRONMENT='DEV'
    window.REACT_APP_NAVBAR_COLOR='LightGreen'
~~~

Mas é melhor vê-lo em ação através do encaminhamento de porta do aplicativo. 

`kubectl port-forward svc/multi-stage-react-app-example 3001:80`

Navegue para http://localhost:3001 e agora você deve ver o ambiente DEV na barra de navegação.

Em um pipeline de entrega contínua, você pode ter duas etapas:
1. criar o configMap com base no arquivo _dev.properties_
2. implantar no cluster de destino com o `kubectl` especificado acima

#### Destruir

`kubectl delete -f kubernetes`

Você pode adotar a mesma abordagem para outros ambientes, como test ou staging.



[^1] **tl;dr** (ou tldr) é uma expressão comum da internet​, uma abreviação das expressões em inglês "too long" e "didn't read". Em português significam "longo demais" e "não li

[^2] **webpack**:  bundler que permite dividir seu código em múltiplos módulos para serem lidos sob demanda