# DesafioPick2024
Desafio de implementar a aplicação  Giropops Senhas em cluster kubernetes e seguro

Objetivo: Criar uma aplicação de gestão de senhas utilizando Docker, Kubernetes, Helm, Kyverno, Cosign, Prometheus, apko e Melange para garantir a segurança, automação, e monitoramento contínuo da aplicação.

Para utilização estou utilizando um cluster k8s na OCI ([oracle cloud](https://cloud.oracle.com/)) na opção de free tier , feito utilizando o projeto do Rapha_Borges https://github.com/Rapha-Borges/oke-free. Com isso possuo um cluster kubernetes no cloud , disponivel para efeutar os estudos e projetos.

Para o projeto do Giropops-senhas , está sendo utilizada a infraesstrutura como pode se observar na imaagem abaixo 

Abaixo temos o projeto dividido em pastas. 

# Prefácio

1 - Infraestrutura - Demonstração da infraestrutura utilizada

2 - [Docker](https://github.com/fellipe85/DesafioPick2024/tree/main/Docker) - Consta a imagem docker e como realizar o build para utilizar uma imagem multistage utilizando chainguard e cosign.

3 - [Melange/Apko](https://github.com/fellipe85/DesafioPick2024/tree/main/melange-apko) - Como gerar a imagem da aplicação utilizando melange e apko . O Pipeline de deploy esta sendo aplicado utilizando github actions e esta localizado em [githuactions](https://github.com/fellipe85/DesafioPick2024/tree/main/.github/workflows)

4 - [Kubernetes](https://github.com/fellipe85/DesafioPick2024/tree/main/Kubernetes) - Os manifestos kubernetes utilizando a imagem gerada pelo melange/apko utilizando cosign e boas praticas para manter a imagem mais segura possivel.
 4.1 - Ingress  

5 -[ Kyverno](https://github.com/fellipe85/DesafioPick2024/tree/main/kyverno) - Todas a Politicas de uso para o projeto estarão aqui

6 - [Locust](https://github.com/fellipe85/DesafioPick2024/tree/main/locust) - Teste de capacidade gerados e seus resultados

7 - [Monitoring ](https://github.com/fellipe85/DesafioPick2024/tree/main/monitoring)- Acesso ao monitoramento da nossa aplicação

8 - [Helm](https://github.com/fellipe85/DesafioPick2024/tree/main/helm_charts)


# 1 - Infraestrutura

Para a infraestrutura, inicialmente foi adquirido um domínio fellipe.dev.br para a criação do projeto. Esse domínio está hospedado no Cloudflare, onde são aplicados diversos módulos de defesa contra DDoS e bots inválidos, oferecendo serviço de WAF para a aplicação. Além disso, temos também a opção de CDN, que faz com que a aplicação responda mais rapidamente em qualquer lugar do mundo. Outra opção ativada é o DNSSEC, que oferece proteção para o domínio. Após o Cloudflare, optei por utilizar um cluster OKE na OCI, pois oferece um cluster sem custo. Para a configuração, foi utilizado o projeto do Rapha_Borges https://github.com/Rapha-Borges/oke-free. Todo o cluster é criado utilizando Terraform ou OpenTofu.

Abaixo está ilustrado como está a infraestrutura.

![image](https://github.com/user-attachments/assets/8dabc021-6872-41ac-90b0-d6733635304d)


# 2 - Docker 
Nesse projeto, optei por utilizar imagens seguras. Em nosso processo inicial, foi criado imagens utilizando da Chainguard, que já estão preparadas com correções de CVEs. Todas as imagens foram criadas em multistage, aproveitando para deixá-las o mais compactas possível.

Abaixo está demonstrado como a imagem é produzida, tanto do Giropops quanto do Redis. Para ambos os casos, foi utilizada a opção de multistage, a fim de diminuir camadas e deixar a imagem o mais segura possível.

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
Após escrever o Dockerfile, foi realizado o build da imagem, enviado para o docker hub e assinado com o cosign. Com isso, a imagem fica armazenada no Docker Hub e assinada. Para realizar o processo de assinar, é necessário ter o cosign instalado na máquina e o login realizado no Docker Hub. Demonstrarei abaixo as etapas do processo: build, instalação do cosign, login no docker hub, push da imagem para o Docker Hub e a assinatura dessa imagem. 

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

Para melhorar a segurança do nosso container e utilizar imagens singlelayer, foi realizado um processo de buildar uma imagem da aplicação utilizando a pasta [giropops-senhas-codigo](https://github.com/fellipe85/DesafioPick2024/tree/main/giropops-senhas-codigo). Nesse processo é feito o build da aplicação e criado uma aplicação do zero, utilizando single layer. Todo esse processo é realizado no githubactions [githuactions](https://github.com/fellipe85/DesafioPick2024/tree/main/.github/workflows). Após o build, é realizado o push para o Docker Hub, que foi o Hub de imagens escolhidos para esse projeto. Um outro detalhe é que esse build pode gerar imagens para todos os tipos de arquitetura de hardware, para este projeto é gerado imagens em amd64 e arm64. Amd64 para quem quiser utilizar em seus pcs uma imagem segura, já o arm64 é a imagem utilizada devido ao cluster kubernetes estar baseado nessa arquitetura.

No Github actions é utilizado o mesmo processo que foi realizado em testes local, a construção da imagem utilizando o melange. Para isso é preciso exportar ao projeto as chaves que são geradas e utilizadas pelo melage/apko.
![image](https://github.com/user-attachments/assets/0b5ef455-5150-45fe-bca9-e84229a4c90a)

Com isso, teremos que apresentar os pacotes necessários para a construção da imagem e temos os passos de build da aplicação, como é possível ver nesse yaml [melange-apko/giropops_alpine_melange.yaml](https://github.com/fellipe85/DesafioPick2024/blob/main/melange-apko/giropops_alpine_melange.yaml)

Após realizar o build nos utilizamos o apko para gerar a imagem 
[melange-apko/giropops_alpine_apko.yaml](https://github.com/fellipe85/DesafioPick2024/blob/main/melange-apko/giropops_alpine_apko.yaml)

Todo esse processo é realizado um build a cada push na main do github. Para o processo de melhoria, podemos realizar quando for atualizado o codigo ou qualquer parte que envolva o melange/apko, fazer scan do código utilizando lint e verificar vunerabilidades com trivy.

O CI/CD está sendo realizado com sucesso como ilustrado abaixo 

![image](https://github.com/user-attachments/assets/04f101d6-3fbb-4985-a0e7-8c814efff70c)
https://github.com/fellipe85/DesafioPick2024/actions/runs/10749261010/job/29813966872

![image](https://github.com/user-attachments/assets/b9e3233b-4aaf-4be0-97c2-55e16e71f61c)

A imagem que está sendo utilizada no nosso kubernetes é essa fellipe85/giropops-senhas-melange-arm:2.0.

# 4 - Kubernetes
Antes de irmos as criações dos manifestos, gostaria de agradecer ao [Rapha-Borges](https://github.com/Rapha-Borges), por ter disponibilizado a criação do cluster Kubernetes na OCI. Não demonstrarei como foi feita minha instalação do cluster, pois utilizei exatamente a documentação criada que se encontra [neste material](https://github.com/Rapha-Borges/oke-free/tree/main). Nele esta sendo explicado como é criada toda a infraestrura na OCI sem custo, possibilitando ter um cluster kubenentes com 3 worknodes baseados em arm.

No processo de instalação do cluster foi necessário instalar a ferramenta do kubectl e oci-cli para comunicação com o cluster. Seguindo para os manifestos, optei por criar um namespace separado para o giropops-senhas e o redis, outro para o locust e outro para monitoramento, segmentando a execução deles somente para cada namespace. Além disso foi criado um ingress para todos os serviços utilizados, cada um representado em seu diretório. Dentro do diretório temos os manifestos para o Giropops-senhas e Redis, ambos necessários para a aplicação funcionar. 

Para instalação utilizamos 

```shell
kubectl apply -f Kubernetes/ 
```

Com esse comando fazemos a instalação da aplicação (giropos e redis) 

Após aplicar a instalação os pods e serviços estão ativos como é possivel observar nas imagens

![image](https://github.com/user-attachments/assets/0a80d619-3d12-494a-a579-62bead5a06fa)
![image](https://github.com/user-attachments/assets/f85a3d95-b8ac-4071-bf4c-0c30e39cdd28)
![image](https://github.com/user-attachments/assets/af5d924e-287c-4135-8fd4-07ace13f0bd0)

Toda e qualquer configuração para o cluster está sendo realizada via manifestos e estará na pasta [Kubernetes](https://github.com/fellipe85/DesafioPick2024/tree/main/Kubernetes)

Após isso seguimos com a instalção do ingress

# 4.1 - Ingress

O ingress tem o intuito de expor a aplicação para a internet utilizado o NLB da OCI no projeto. Para realizar a instalação utilizamos o helm 
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
E aplicamos a configuração do ingress que está na pasta [Kubernetes/ingress](https://github.com/fellipe85/DesafioPick2024/tree/main/Kubernetes/ingress)

```shell
Kubectl apply -f Kubernetes/igress/senhas-ingress.yaml
```
Após a instalação, podemos acessar nossa aplicação na página [https://senhas.fellipe.dev.br](https://senhas.fellipe.dev.br)

![image](https://github.com/user-attachments/assets/f8b7a1a6-c483-49f6-b463-d2693c9b3a8d)

E podemos observar que o certificado configurado nos manifestos do ingress foram aplicados e está sendo utilizado.

![image](https://github.com/user-attachments/assets/688fa228-416e-4cf6-a5ae-4ccf6a7a2e03)

Podemos também observar no cloudflare o trafego sendo utilizado a maioria pelo CDN, sem a necessidade de utilizar muito hardware.

![image](https://github.com/user-attachments/assets/29de1bad-41c1-4195-9d4c-3bad25a956d9)



=======
# 5 - Kyverno - Politicas de uso para o projeto

#### Instalando o Kyverno

Para instalar é muito simples, vamos fazer através do helm

Primeiro passo instalar o helm
```bash
helm repo add kyeverno https://kyverno.github.io/kyverno
helm repo update
helm install kyverno kyverno/kyverno --namespace kyverno --create-namespace
```

Para ter rodando em alta disponibilidade o Admission controller precisa rodar em 3 nós. No site do kyverno existe diversas policies já prontas.

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

Para esta aplicação, criamos uma regra para que sempre tenha que existir um valor de cpu e memoria definido e também containers com root em execução não serão criados.

Caso queira adicionar mais regras e saber mais sobre o assunto, a [documentação](https://kyverno.io/policies/) do kyverno possui tudo muito bem detalhado, inclusive para melhorias no futuro seria aplicar mais policies deixando mais seguro o cluster.

# 6 - Locust - Teste de capacidade gerados e seus resultados 

O Locust é uma ferramenta OpenSource que permite executarmos estresse de carga na aplicação. Com ela, é possivel criar scripts em python para definir quais paths serão testados e o tipo de teste a ser feito no website. Você pode ver como o Locust está provisionado nos manifestos da pasta [Locust](https://github.com/fellipe85/DesafioPick2024/tree/main/locust) 

Foi criado dois testes: 
- Gerar senhas diversas na API a partir da rota /api/gerar-senha
- Consultar as senhas através da rota /api/senhas

Para acessar o locust acessar a url https://locust.fellipe.dev.br

Através da interface web, o Locust acessará diretamente no Service do Giropops-Senhas. Como os dois serviços estão dentro do Cluster, foi feito pelo DNS interno para que tenha problemas de carga na máquina local ou ALB:

![image](https://github.com/user-attachments/assets/80aa1682-529c-4522-a39e-59d554c5d639)

No inicio do teste podemos observar que temos apenas 3 pods para giropops senhas

![image](https://github.com/user-attachments/assets/19b2e690-8879-40b9-8403-aa12ba71e840)

Podemos observar os testes em execução

![image](https://github.com/user-attachments/assets/0e9fa282-362b-426b-8280-e5b63571220e)

Como no caso temos camada de CDN para a nossa aplicação os pods se permaneceram intactos.

Desabilitadno o CDN, podemos observar que o HPA entrou e fez o update para os 10 pods

![image](https://github.com/user-attachments/assets/2e99be77-842f-4492-b7bd-66fb3dbb33a5)


Após baixar a carga, é possível notar que os pods terminam e a aplicação volta ao seu estado de 3 pods 

![image](https://github.com/user-attachments/assets/1968983c-f383-414b-a103-565952bb1d56)

Podemos observar que com o CDN Ativo tivemos carga de 10 mil usuários sem nenhum erro ou perda de pacotes, melhorando bastante a questão de utilização de recursos no cluster.  Os resultados do teste  podem ser observados na pasta [locust/Resultados](https://github.com/fellipe85/DesafioPick2024/tree/main/locust/resultados)


# 7 - Monitoring - 

Para monitoramento esscolhemos a stack Prometheus/Grafana/Alertmanager para realizar o monitoramento do cluster e da aplicação. Para começar precisamos realizar a instalação e para isso vamos realizar atraves do kube-prometheus, ele ja tem node_exporter , prometheus , grafana , tudo integrado 

O kube-prometheus é um conjunto de manifestos do Kubernetes que nos permite ter o Prometheus Operator, Grafana, AlertManager, Node Exporter, Kube-State-Metrics, Prometheus-Adapter instalados e configurados de forma tranquila e com alta disponibilidade. Além disso, ele nos permite ter uma visão completa do nosso cluster de Kubernetes. Ele nos permite monitorar todos os componentes do nosso cluster de Kubernetes, como por exemplo: kube-scheduler, kube-controller-manager, kubelet, kube-proxy, etc.

#### Instalando o Grafana/Prometheus/AlertManager

Para instalar só seguir com os comandos 
```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
O processo de instalação pode demorar um pouco , podemos checar se foi instalado com o comando 
```shell
kubectl get servicemonitors -A
```
Após a instalação dos CRDs, vamos instalar o Prometheus e o Alertmanager. Para isso, basta executar o seguinte comando:
```shell
helm install prometheus prometheus-community/prometheus --namespace monitoring
```
Checar se foi instalado e esta rodando 

```shell
kubectl get pods -n monitoring
```
Podemos ver o prometheus rodando , apos isso vamos criar o service e o ingress para o prometheus.

Para acessar o Grafana, vamos utilizar o usuário `admin` e a senha `admin`, e já no primeiro login ele irá pedir para você alterar a senha.No Grafana para monitoramento do cluster vamos importar o dashboard do GrafanaLabs (https://grafana.com/grafana/dashboards/12740)



# 8 - Helm

