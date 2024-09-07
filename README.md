# DesafioPick2024
Desafio de implementar a aplicação  Giropops Senhas em cluster kubernetes e seguro

Objetivo: Criar uma aplicação de gestão de senhas utilizando Docker, Kubernetes, Helm, Kyverno, Cosign, Prometheus, apko e Melange para garantir a segurança, automação, e monitoramento contínuo da aplicação.

O projeto está dividio em pastas, em cada pasta exista a explicação do seu funcionamento. 

Prefácio

1 - Infraestrutura 

2 - [Docker](https://github.com/fellipe85/DesafioPick2024/tree/main/Docker) - Consta a imagem docker e como realizar o build para utilizar uma imagem multistage utilizando chainguard e cosign.

3 - [Melange/Apko](https://github.com/fellipe85/DesafioPick2024/tree/main/melange-apko) - Como gerar a imagem da aplicação utilizando melange e apko . O Pipeline de deploy esta sendo aplicado utilizando github actions e esta localizado em [githuactions](https://github.com/fellipe85/DesafioPick2024/tree/main/.github/workflows)

4 - [Kubernetes](https://github.com/fellipe85/DesafioPick2024/tree/main/Kubernetes) - Os manifestos kubernetes utilizando a imagem gerada pelo melange/apko utilizando cosign e boas praticas para manter a imagem mais segura possivel.

5 - Kyverno - Todas a Politicas de uso para o projeto estarão aqui

6 - Locust - Teste de capacidade gerados e seus resultados

7 - Monitoring - Acesso ao monitoramento da nossa aplicação


1 - Infraestrutura

Para a infraestrutura , incialmente foi adquirido um dominio fellipe.dev.br para criação do projeto , esse dominio esta hospedado no Cloudflare , onde é aplicada diversos modulos de defesa contra DDOs , bots inválidos oferencendo serviço de WAF para a aplicação , além disso temos também a opção de CDN , fazendo com que a aplicação responda mais rapidamente em qualquer lugar do mundo, outra opção que está ativada é DNSSEC , oferecendo proteção para o dominio. Após o cloudflare , optei em utilizar um cluster OKE na OCI , pois oferece um cluster sempre de graça. Para a configuração foi utilizada o projeto do Rapha_Borges https://github.com/Rapha-Borges/oke-free. Todo o cluster é criado utilizando terraform ou opentofu. 

Abaixo está ilustrado como está a infraestrutura.

2 - Docker 

Nesse projeto escolhemos utilizar imagens seguras , em nosso processo inicial fizemos imagens utilizando imagens da chainguard que já estão preparadas com correções de CVEs . Todas imagens foram criadas em Multistage aproveitando para deixar o mais compacta possivel.


3 - Melange/Apko

Para melhorar a segurança do nosso container e utilizar imagens singlelayer , estamos realizando um processo de de buildar uma imagem da aplicação utilizando a pasta giropops-senhas-codigo , nesse processo estamos fazendo o build da aplicação e criando uma aplicação do zero utilizando single layer . Todo esse processo é realizado no githubactions [githuactions](https://github.com/fellipe85/DesafioPick2024/tree/main/.github/workflows).Após o build é realizado o push para o Docker Hub , que foi o Hub de imagens escolhidos para esse projeto.

3 - Kubernetes

Optei por criar um namespace separado para o giropops-senhas e o redis , nesse caso , separando a execução deles somente para esse namespace. Além disso foi criado um ingress para todos os serviços utilizados , cada um representado em seu diretorio. Dentro do diretorio temos os manifestos para o Giropops-senhas e Redis , ambos ncessários para a aplicação funcionar. 

Para instalação utilizamos 

```shell
kubectl apply -f Kubernetes/ 
```

Com esse comando fazemos a instalação da aplicação(giropos e redis) 
