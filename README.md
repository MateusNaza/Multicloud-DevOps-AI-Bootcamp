# Multicloud-DevOps-AI-Bootcamp
     
# Situação     
     
Este projeto tem como objetivo simular um cenário real em que uma empresa, ao perceber que seu concorrente está oferecendo preços mais competitivos, busca maneiras de reduzir seus custos para se manter competitiva no mercado.    
     
Uma das soluções  encontradas foi diminuir o custo de __Suporte ao Cliente__, pois os gastos mensais com folha de pagamento estavam em __R$ 2,5 milhões__. Com a aplicação da solução apresentada abaixo, estima-se que o custo mensal cairá para __R$ 250 mil__, resultando em uma __diminuição de R$ 2,25 milhões nos custos operacionais__, o que impactará no custo final do produto, tornando-o mais competitivo.
     
# Solução     
     
Para obtermos esse retorno, será necessário rearquitetar e migrar a aplicação para rodar de forma moderna na nuvem com DevOps integrada com Assistentes de IA.
     
![arquitetura](assets/arquitetura.png)
    
    
# Dia 1
     
## Preparação de ambiente
    
Para esse desafio, estaremos utilizando um ambiente em núvem, dentro de uma instância EC2 da AWS. Os primeiros passos serão a configuração desse ambiente.
    
1. Criar de uma 'Role' para o EC2.
2. Lançar a instância EC2
3. Conectar à instância
    
![instancia conectada](assets/instancia_conectada.png)
     
Agora vamos configurar o ambiente fazendo as instalações necessárias
     
```bash
sudo yum update -y
sudo yum install -y yum-utils

sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo

sudo yum -y install terraform
``` 
     
## Iniciando com Terraform
     
Para dar os primeiros passos com _Terraform_ criei o primeiro arquivo que faz a criação de um bucket S3. Também tive o primeiro contato com os principais comandos:
     
```bash
#Inicialize o Terraform
terraform init

#Revise o plano
terraform plan

# Aplica a configuração
terraform apply

# Comando para destruir os recursos recem criados
terraform destroy
```
      
![bucket criado](assets/bucket_criado.png)
     

# Dia 2
     
## Criação de Tabelas no _DynamoDB_
      
Logo no início do segundo dia, já consegui perceber o poder e a velocidade do Terraform para provisionar recursos na nuvem.

Alterei o arquivo de configuração para criar três tabelas no _DynamoDB_, e o processo levou apenas alguns segundos para ser concluído. O que mais me impressionou foi a escalabilidade, pois, como a criação das tabelas foi realizada em paralelo, isso significa que, mesmo se fosse necessário criar 100 tabelas, o tempo teóricamente para a execução seria o mesmo ou próximo desse.
     
![tabelas dynamodb](assets/tabelas_dynamodb.png)
          
## Preparando o Docker
     
Para iniciar com o Docker, primeiro realizei os comandos de instalação e configuração.
     
```bash
sudo yum update -y
sudo yum install docker -y

sudo systemctl start docker
sudo systemctl enable docker

sudo usermod -a -G docker $(whoami)
newgrp docker
```
     
## Configurando Backend

Com o Docker instalado, podemos baixar a imagem do _CloudMart_ e configurar o container do _backend_.

```bash
mkdir backend && cd backend

wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-backend.zip
unzip cloudmart-backend.zip

nano .env

# PORT=5000
# AWS_REGION=us-east-1

nano Dockerfile

# FROM node:18
# WORKDIR /usr/src/app
# COPY package*.json ./
# RUN npm install
# COPY . .
# EXPOSE 5000
# CMD ["npm", "start"]

docker build -t cloudmart-backend .
docker run -d -p 5000:5000 --env-file .env cloudmart-backend
```     
            
## Configurando Frontend
      
Agora criaremos também a imagem para rodar o Frontend da aplicação.
      
```bash
mkdir frontend && cd frontend

wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-frontend.zip
unzip cloudmart-frontend.zip

nano .env

# VITE_API_BASE_URL=http://<seu-ip-ec2>:5000/api

nano Dockerfile

# FROM node:16-alpine as build
# WORKDIR /app
# COPY package*.json ./
# RUN npm ci
# COPY . .
# RUN npm run build

# FROM node:16-alpine
# WORKDIR /app
# RUN npm install -g serve
# COPY --from=build /app/dist /app
# ENV PORT=5001
# ENV NODE_ENV=production
# EXPOSE 5001
# CMD ["serve", "-s", ".", "-l", "5001"]

docker build -t cloudmart-frontend .
docker run -d -p 5001:5001 cloudmart-frontend
```
     
>Precisei também liberar as portas 5000 e 5001 no security group para conseguir acessar a aplicação
     
A Aplicação ficou disponível no link 'http://{ip-publico-ec2}:5001' e para cadastrar produtos basta entrar na parte de administrador 'http://{ip-publico-ec2}5001/admin'.
      
![produtos cloudmart](assets/produtos_cloudmart.png)
     

# Dia 3 - Parte 1
     
## Kubernetes Configurações iniciais
     
Hoje iremos iniciar a utilização do kubernetes, acessando atravéz do serviço Amazon EKS, esse serviço é pago e deve-se ter muita atenção ao utilizá-lo para não esquece-lo aberto.
     
Primeiro eu criei um novo usuário IAM chamado 'eksuser' e depois dentro da minha instância EC2 segui os seguintes passos:
     
```bash
# Acessei o usuário eksuser
aws configure

# Instala ferramenta eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo cp /tmp/eksctl /usr/bin
eksctl version

# Instala ferramenta kubectl
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version --short --client

# Cria Cluster Kubernetes
eksctl create cluster \
  --name cloudmart \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 1 \
  --with-oidc \
  --managed
```
    
Após o último comando, irá iniciar algumas stacks do CloudFomation, pois é atravéz dele que são criados todos os recursos necessários do EKS.
     
![CloudFormation](assets/CloudFormation.png)
     

## Kubernetes primeiros passos
     
Agora que temos nosso cluster kubernetes ativo, podemos iniciar com os primeiros comandos.
      
```bash
# Conecta ao cluster
aws eks update-kubeconfig --name cloudmart

# Verifica conectividade do Cluster
kubectl get svc
kubectl get nodes

# Dá permissão (atravéz de Role) ao cluster para acessar outros serviços AWS
eksctl create iamserviceaccount \
  --cluster=cloudmart \
  --name=cloudmart-pod-execution-role \
  --role-name CloudMartPodExecutionRole \
  --attach-policy-arn=arn:aws:iam::aws:policy/AdministratorAccess\
  --region us-east-1 \
  --approve
```
     
Agora precisamos registrar as imagens do Backend e do Frontend no ECR. Para isso seguimos os seguintes passos:
      
1- Abra o serviço ECS no console AWS
2- Vá na parte de repositórios publicos
3- Crie um novo repositório (cloudmart-backend e cloudmart-frontend)
4- Clique no repositório de depois em 'View Push Commands', e faça, dentro da instância, os comandos apresentados
      
## Deployment do Backend no Kubernetes
      
```bash
# Entre na pasta do backend
cd backend

# Cria um novo deployment do Kubernetes no arquivo yaml
nano cloudmart-backend.yaml

# Realiza o Deployment do Backend no Kubernetes
kubectl apply -f cloudmart-backend.yaml

# Comandos para acompanhar a criação dos objetos
kubectl get pods
kubectl get deployment
kubectl get service # Lembre-se de copiar o IP Público para usar no .env do Frontend
```
       
## Deployment do Frontend no Kubernetes
      
```bash
# Entre na pasta do frontend
cd frontend

# Edita o arquivo .env
nano .env

# VITE_API_BASE_URL=http://a75602f93a86f47ca977bae4e78d0cde-1631225860.us-east-1.elb.amazonaws.com:5000/api

# Cria um novo deployment do Kubernetes no arquivo yaml
nano cloudmart-frontend.yaml

# Realiza o Deployment do frontend no Kubernetes
kubectl apply -f cloudmart-frontend.yaml
```
     
E pronto, temos a aplicação rodando dentro dos containeres orquestrados pelo EKS, agora para não gerar cobranças exessivas vamos fazer a limpeza do ambiente.
      
```bash
kubectl delete service cloudmart-frontend-app-service
kubectl delete deployment cloudmart-frontend-app
kubectl delete service cloudmart-backend-app-service
kubectl delete deployment cloudmart-backend-app

eksctl delete cluster --name cloudmart --region us-east-1
```
      
# Dia 3 - Parte 2
      
## Configurando Pipeline CI/CD
     
Para essa etapa continuaremos rodando a aplicação dentro do Kubernetes, porém faremos uma esteira de CI/CD para a parte de Frontend para que todas as mudanças feitas no código e atualizadas no repositório github sejam refletidas na aplicação em produção de forma continua.
      
![CICD.png](assets/CICD.png)
      
>Como a pipeline CI/CD será aplicada somente na parte de frontend do projeto, subi um novo repositório no Github contendo o código fonte somente do front num repositório >chamado __Cloudmart__.     

## AWS CodePipeline   
     
Para configurar o _CodePipeline_ segui os seguintes passos:    
     
1- No console cliquei no botão __Create new Pipeline__ .          
2- Selecionei a opção __Build Custom Pipeline__.   
       
![custom_pipeline](assets/custom_pipeline.png)     
      
3- Nomeei a pipeline como: __cloudmart-cicd-pipeline__.     
4- Conectei meu repositório Github seguindo os passos do console.      
       
![conection_github](assets/conection_github.png)     
     
5- Configurei a etapa de build para utilizar o __CodeBuild__ e já cliquei para criar um projeto dentro desse serviço.     
      
![codebuild_option](assets/codebuild_option.png)     
     
6- Na aba __CodeBuild__, nomeeio meu projeto de __cloudmart-application__ e fiz as seguintes alterações.     
     
![codebuild_enviroment.png](assets/codebuild_enviroment.png)     
      
7- Ainda nessa guia, adicionei uma variáveu de ambiente com: KEY=ECR_REPO e VALUE=URI-do-Repositório-ECR.     
8- Criei uma _buildspec.yaml_ com o seguinte conteúdo.     
      
```bash
version: 0.2
phases:
  install:
    runtime-versions:
      docker: 20
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - REPOSITORY_URI=$ECR_REPO
      - aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/r2l2v5m8
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - export imageTag=$CODEBUILD_RESOLVED_SOURCE_VERSION
      - printf '[{\"name\":\"cloudmart-app\",\"imageUri\":\"%s\"}]' $REPOSITORY_URI:$imageTag > imagedefinitions.json
      - cat imagedefinitions.json
      - ls -l

env:
  exported-variables: ["imageTag"]

artifacts:
  files:
    - imagedefinitions.json
    - cloudmart-frontend.yaml
```
      

E pronto, após esse comando criamos nossa pipeline.
     
![pipeline_cicd](assets/pipeline_cicd.png)    
     

> Detalhe importante!!!
> Deve-se dar a permissão para acessar o ECR para a role 'codebuild...' no IAM
       
## Adicionando processo de Deploy
      
Após concluir a pipeline, adicionei mais um processo a ela, e na ação que inseri criei outro projeto do _CodeBuild_ seguindo os mesmos passos anteriores porém com as seguintes modificações:     
  - O nome do projeto foi __cloudmartDeployToProduction__.
  - Coloquei de variáveis de ambiente a AWS_ACCESS_KEY_ID e AWS_SECRET_ACCESS_KEY.
       
Para o _buildspec.yaml_ desse arquivo vamos usar o seguinte código:      
```bash
version: 0.2

phases:
  install:
    runtime-versions:
      docker: 20
    commands:
      - curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
      - chmod +x ./kubectl
      - mv ./kubectl /usr/local/bin
      - kubectl version --short --client
  post_build:
    commands:
      - aws eks update-kubeconfig --region us-east-1 --name cloudmart
      - kubectl get nodes
      - ls
      - IMAGE_URI=$(jq -r '.[0].imageUri' imagedefinitions.json)
      - echo $IMAGE_URI
      - sed -i "s|CONTAINER_IMAGE|$IMAGE_URI|g" cloudmart-frontend.yaml
      - kubectl apply -f cloudmart-frontend.yaml
```
       
## Edição de arquivo .yaml no github     
       
A útima configuração foi apenas alterar o caminho da imagem pela variável CONTAINER_IMAGE dentro do meu arquivo '.yaml' no repositório git do projeto. Após fazer o push dessa alteração o meu pipeline foi automaticamente acionado e efetuado todo o processo automático de deploy da alteração.      
       
![pipeline_cicd_completa](assets/pipeline_cicd_completa.png)     
     
## Testando alterações e subindo para produção
      
Vou fazer um teste simples alterando um texto para verificar se as alterações estão subindo para produção como deveriam.
     
### Antes da alteração

![antes_alteracao.png](assets/antes_alteracao.png)

Somente alterei um texto e dei o push do repositório na branch main...
       
### Depois da Alteração
      
![depois_mudanca](assets/depois_mudanca.png)
       
