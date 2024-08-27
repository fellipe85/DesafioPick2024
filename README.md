# DesafioPick2024
Desafio de implementar a aplicação  Giropops Senhas em cluster kubernetes e seguro

Objetivo: Criar uma aplicação de gestão de senhas utilizando Docker, Kubernetes, Helm, Kyverno, Cosign, Prometheus, apko e Melange para garantir a segurança, automação, e monitoramento contínuo da aplicação.

O projeto está dividio em pastas, em cada pasta exista a explicação do seu funcionamento. 

1 - [Docker](https://github.com/fellipe85/DesafioPick2024/tree/main/Docker) - Consta a imagem docker e como realizar o build para utilizar uma imagem multistage utilizando chainguard

2 - [Kubernetes](https://github.com/fellipe85/DesafioPick2024/tree/main/Kubernetes) - Os manifestos kubernetes utilizando a imagem gerada pelo melange/apko.

3 - [Melange/Apko](https://github.com/fellipe85/DesafioPick2024/tree/main/melange-apko) - Como gerar a imagem da aplicação utilizando melange e apko . O Pipeline de deploy esta sendo aplicado utilizando github actions e esta localizado em [githuactions](https://github.com/fellipe85/DesafioPick2024/tree/main/.github/workflows)
