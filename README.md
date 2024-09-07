# DesafioPick2024
Desafio de implementar a aplicação  Giropops Senhas em cluster kubernetes e seguro

Objetivo: Criar uma aplicação de gestão de senhas utilizando Docker, Kubernetes, Helm, Kyverno, Cosign, Prometheus, apko e Melange para garantir a segurança, automação, e monitoramento contínuo da aplicação.

Para utilização estou utilizando um cluster k8s na OCI ([oracle cloud](https://cloud.oracle.com/)) na opção de free tier , feito utilizando o projeto do Rapha_Borges https://github.com/Rapha-Borges/oke-free. Com isso possuo um cluster kubernetes no cloud , disponivel para efeutar os estudos e projetos.

Para o projeto do Giropops-senhas , está sendo utilizada a infraesstrutura como pode se observar na imaagem abaixo 

Abaixo temos o projeto dividido em pastas. 

# Prefácio

1 - Infraestrutura 

2 - [Docker](https://github.com/fellipe85/DesafioPick2024/tree/main/Docker) - Consta a imagem docker e como realizar o build para utilizar uma imagem multistage utilizando chainguard e cosign.

3 - [Melange/Apko](https://github.com/fellipe85/DesafioPick2024/tree/main/melange-apko) - Como gerar a imagem da aplicação utilizando melange e apko . O Pipeline de deploy esta sendo aplicado utilizando github actions e esta localizado em [githuactions](https://github.com/fellipe85/DesafioPick2024/tree/main/.github/workflows)

4 - [Kubernetes](https://github.com/fellipe85/DesafioPick2024/tree/main/Kubernetes) - Os manifestos kubernetes utilizando a imagem gerada pelo melange/apko utilizando cosign e boas praticas para manter a imagem mais segura possivel.
 4.1 - Ingress  

5 -[ Kyverno](https://github.com/fellipe85/DesafioPick2024/tree/main/kyverno) - Todas a Politicas de uso para o projeto estarão aqui

6 - [Locust](https://github.com/fellipe85/DesafioPick2024/tree/main/locust) - Teste de capacidade gerados e seus resultados

7 - [Monitoring ](https://github.com/fellipe85/DesafioPick2024/tree/main/monitoring)- Acesso ao monitoramento da nossa aplicação

8 - [Helm](https://github.com/fellipe85/DesafioPick2024/tree/main/helm_charts)


# 1 - Infraestrutura

Para a infraestrutura , incialmente foi adquirido um dominio fellipe.dev.br para criação do projeto , esse dominio esta hospedado no Cloudflare , onde é aplicada diversos modulos de defesa contra DDOs , bots inválidos oferencendo serviço de WAF para a aplicação , além disso temos também a opção de CDN , fazendo com que a aplicação responda mais rapidamente em qualquer lugar do mundo, outra opção que está ativada é DNSSEC , oferecendo proteção para o dominio. Após o cloudflare , optei em utilizar um cluster OKE na OCI , pois oferece um cluster sempre de graça. Para a configuração foi utilizada o projeto do Rapha_Borges https://github.com/Rapha-Borges/oke-free. Todo o cluster é criado utilizando terraform ou opentofu. 

Abaixo está ilustrado como está a infraestrutura.

# 2 - Docker 

Nesse projeto escolhemos utilizar imagens seguras , em nosso processo inicial fizemos imagens utilizando imagens da chainguard que já estão preparadas com correções de CVEs . Todas imagens foram criadas em Multistage aproveitando para deixar o mais compacta possivel.

Abaixo demonstramos como esta sendo produzida a imagem tanto do giropops senhas. Para ambos os casos foi utilizado a opção de multistage afim de diminuir camadas e deixar a imagem a o mais segura possivel.
```giropops-senhas.yaml
FROM cgr.dev/chainguard/python:latest-dev as build
WORKDIR /app
COPY . ./
RUN pip install -r requirements.txt --user


FROM cgr.dev/chainguard/python:latest
WORKDIR /app
COPY --from=build /FROM cgr.dev/chainguard/redis
COPY --from=build /app /app
EXPOSE 5000
ENTRYPOINT ["/home/nonroot/.local/bin/flask","run","--host=0.0.0.0"]
#ENTRYPOINT ["python" , "app.py"]
```
Após escrever o Dockerfile , vamos realizar o build da imagem , enviar  a imagem para o docker hub e assinar com o cosign. Com isso teremos nossa imagem armazenada no Hub e assinada. Para realizar devemos ter o cosgin instalado na máquina e o login realizado no dockerhub , abaixo vamos demonstrar como é feita a build , instalação do cosign , login no docker hub , push da imagem para o hub e assinatura dessa imagem 

```shell
cd Docker
docker build -t fellipe85/giropops-senhas:1.0 .
docker login 
#insira os dados para login
docker push fellipe85/giropops-senhas:1.0
```
Instalando o cosign
```shell 
# binary
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign
cosgin generate-key-pair
cosign sign -key cosign.key fellipe85/giropops-senhas-melange:1.0 -y
```
Com isso temos nossa imagem assinada no dockerhub

![image](https://github.com/user-attachments/assets/6e28d08c-0f4f-45d7-82ba-515f67d9867e)



# 3 - Melange/Apko

Para melhorar a segurança do nosso container e utilizar imagens singlelayer , estamos realizando um processo de de buildar uma imagem da aplicação utilizando a pasta giropops-senhas-codigo , nesse processo estamos fazendo o build da aplicação e criando uma aplicação do zero utilizando single layer . Todo esse processo é realizado no githubactions [githuactions](https://github.com/fellipe85/DesafioPick2024/tree/main/.github/workflows).Após o build é realizado o push para o Docker Hub , que foi o Hub de imagens escolhidos para esse projeto. Um outro detalhe é que esse build pode gerar imagens para todos os tipos de arquitetura de hardware , no caso do meu projeto estou gerando imagens em amd64 e arm64, amd64 para quem quiser utilizar em seus pcs uma imagem segura e o arm64 é a imagem que estou utilizado devido ao meu cluster kubernetes estar baseado nessa arquitetura

no Github actions estamos usando o mesmo processo que foi realizado em testes na minha máquina, fazemos a construção da imagem utilizando o melange , para isso precisamos exportar ao projeto as chaves que são geradas e utilizadas pelo melage/apko .
![image](https://github.com/user-attachments/assets/0b5ef455-5150-45fe-bca9-e84229a4c90a)

e nisso teremos que apresentar os pacotes necessários para a construção da imagem e além disso temos os passos de build da aplicação, como é possivem ver nesse yaml [melange-apko/giropops_alpine_melange.yaml](https://github.com/fellipe85/DesafioPick2024/blob/main/melange-apko/giropops_alpine_melange.yaml)

Após realizar o build nos utilizamos o apko para gerar a imagem 
[melange-apko/giropops_alpine_apko.yaml](https://github.com/fellipe85/DesafioPick2024/blob/main/melange-apko/giropops_alpine_apko.yaml)

Todo esse processo é realizado um build a cada push na main do github actions. Para melhoria podemos realizar somente quando for atualizado o codigo ou qualquer parte que envolva o melange/apko e além disso fazer scan do código utilizando lint.

O CI/CD está sendo realizado com sucesso como pode observar abaixo 

![image](https://github.com/user-attachments/assets/04f101d6-3fbb-4985-a0e7-8c814efff70c)
https://github.com/fellipe85/DesafioPick2024/actions/runs/10749261010/job/29813966872

![image](https://github.com/user-attachments/assets/b9e3233b-4aaf-4be0-97c2-55e16e71f61c)

A imagem que está sendo utilizada no nosso kubernetes é essa fellipe85/fellipe85/giropops-senhas-melange-arm:2.0 , tive que optar por ser arm devido ao cluster kubernetes estar em arquitetura arm.

# 4 - Kubernetes

Optei por criar um namespace separado para o giropops-senhas e o redis , nesse caso , separando a execução deles somente para esse namespace. Além disso foi criado um ingress para todos os serviços utilizados , cada um representado em seu diretorio. Dentro do diretorio temos os manifestos para o Giropops-senhas e Redis , ambos ncessários para a aplicação funcionar. 

Para instalação utilizamos 

```shell
kubectl apply -f Kubernetes/ 
```

Com esse comando fazemos a instalação da aplicação(giropos e redis) 

Após aplicar a instalação os pods e serviços estão ativos como é possivel observar na imagem 


Após isso seguimos com a instalção do ingress

# 4.1 - Ingress

O ingress tem o intuito de expor nossa aplicação para a internet utilizado o NLB da OCI para o nosso projeto nesse caso , para realizar a instalação utilizamos o helm 
```shell
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.annotations."oci\.oraclecloud\.com/load-balancer-type"=nlb \
  --set controller.service.annotations."oci-network-load-balancer\.oraclecloud\.com/security-list-management-mode"=All \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```
E aplicamos a configuração do ingress que está na pasta Kubernetes/igress

```shell
Kubectl apply -f Kubernetes/igress/senhas-ingress.yaml
```
Após a instalação , podemos acessar nossa aplicação na página . Para o meu caso foi criada a url senhas.fellipe.dev.br

![image](https://github.com/user-attachments/assets/f8b7a1a6-c483-49f6-b463-d2693c9b3a8d)

E podemos observar também que o certificado configurado em nossos manifestos do ingress foram aplicados e estão sendo utilizados 

![image](https://github.com/user-attachments/assets/688fa228-416e-4cf6-a5ae-4ccf6a7a2e03)


=======
# 5 - Kyverno - Politicas de uso para o projeto

#### Instalando o Kyverno

Para instalar é muito simples  , vamos fazer atraves do helm

Primeiro passo instalar o helm
```bash
helm repo add kyeverno https://kyverno.github.io/kyverno
helm repo update
helm install kyverno kyverno/kyverno --namespace kyverno --create-namespace
```

Para ter rodando em alta disponibilidade o Admission controller precisa rodar em 3 nós , no site do kyverno tem bastante policies ja prontas.

Verificando se foi instalado 

- **Verifique os Pods:**
    
    ```
    kubectl get pods -n kyverno
    ```
    
    Este comando deve mostrar os pods do Kyverno em execução no namespace especificado.
    
- **Verifique os CRDs:**
    
    ```
    kubectl get crd | grep kyverno
    ```

Para a nossa aplicação , criamos uma regra para que sempre tenha que existir um valor de cpu e memoria definido , e também containers sem  root em execução.


# 6 - Locust - Teste de capacidade gerados e seus resultados 

O Locust é uma ferramenta OpenSource que permite executarmos  estresse  de carga em nossa aplicação. Com ela, é possivel criar  scripts python para definir quais paths serão testamos e o tipo de teste a ser feito no website Você pode ver como o meu Locust está provisionado nos manifestos da pasta Locust 

Esta dividida em dois testes 
Ficar gerando senhas na API a partir da rota /api/gerar-senha
Ficar consultando as senhas através da rota /api/senhas

Para acessar o locust acessar a url https://locust.fellipe.dev.br

Através da interface web, vou fazer o Locust bater diretamente no Service do Giropops-Senhas. Como os dois serviços estão dentro do Cluster, farei isso pelo DNS interno para que eu não tenha problemas de carga na minha máquina ou ALB:

![image](https://github.com/user-attachments/assets/80aa1682-529c-4522-a39e-59d554c5d639)

No inicio do teste podemos observar   que temos apenas 3 pods para giropops senhas

![image](https://github.com/user-attachments/assets/19b2e690-8879-40b9-8403-aa12ba71e840)

Podemos observar os testes em execução

![image](https://github.com/user-attachments/assets/0e9fa282-362b-426b-8280-e5b63571220e)

Como no caso temos camada de CDN para a nossa aplicação os pods se permaneceram intacticos .

 Desabilitadno o CDN , podemos observar que o HPA entrou e fez o update para os 10 pods

![image](https://github.com/user-attachments/assets/2e99be77-842f-4492-b7bd-66fb3dbb33a5)


Após baixar a carga , observamos os pods morrendo e voltando ao seu estado de 3 pods para a aplicação 

![image](https://github.com/user-attachments/assets/1968983c-f383-414b-a103-565952bb1d56)

Podemos observar que com o CDN Ativo tivemos carga de 10 mil usuários sem nenhum erro ou perda de pacotes , melhorando bastante a questão de utilização de recursos no cluster.  Os resultados do teste  podem ser observados na pasta [locust/Resultados](https://github.com/fellipe85/DesafioPick2024/tree/main/locust/resultados)








# 7 - Monitoring - 

# 8 - Helm

