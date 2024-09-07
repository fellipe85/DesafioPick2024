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

5 - Kyverno - Todas a Politicas de uso para o projeto estarão aqui

6 - Locust - Teste de capacidade gerados e seus resultados

7 - Monitoring - Acesso ao monitoramento da nossa aplicação


# 1 - Infraestrutura

Para a infraestrutura , incialmente foi adquirido um dominio fellipe.dev.br para criação do projeto , esse dominio esta hospedado no Cloudflare , onde é aplicada diversos modulos de defesa contra DDOs , bots inválidos oferencendo serviço de WAF para a aplicação , além disso temos também a opção de CDN , fazendo com que a aplicação responda mais rapidamente em qualquer lugar do mundo, outra opção que está ativada é DNSSEC , oferecendo proteção para o dominio. Após o cloudflare , optei em utilizar um cluster OKE na OCI , pois oferece um cluster sempre de graça. Para a configuração foi utilizada o projeto do Rapha_Borges https://github.com/Rapha-Borges/oke-free. Todo o cluster é criado utilizando terraform ou opentofu. 

Abaixo está ilustrado como está a infraestrutura.

# 2 - Docker 

Nesse projeto escolhemos utilizar imagens seguras , em nosso processo inicial fizemos imagens utilizando imagens da chainguard que já estão preparadas com correções de CVEs . Todas imagens foram criadas em Multistage aproveitando para deixar o mais compacta possivel.


# 3 - Melange/Apko

Para melhorar a segurança do nosso container e utilizar imagens singlelayer , estamos realizando um processo de de buildar uma imagem da aplicação utilizando a pasta giropops-senhas-codigo , nesse processo estamos fazendo o build da aplicação e criando uma aplicação do zero utilizando single layer . Todo esse processo é realizado no githubactions [githuactions](https://github.com/fellipe85/DesafioPick2024/tree/main/.github/workflows).Após o build é realizado o push para o Docker Hub , que foi o Hub de imagens escolhidos para esse projeto.

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
# 4 - Kyverno - Politicas de uso para o projeto

# 5 - Locust - Teste de capacidade gerados e seus resultados 

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

Podemos observar que com o CDN Ativo tivemos carga de 10 mil usuários sem nenhum erro ou perda de pacotes , melhorando bastante a questão de utilização de recursos no cluster.  Os resultados do teste  podem ser observados na pasta locust/Resultados








# 6 - Monitoring - 
