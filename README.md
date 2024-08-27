# DesafioPick2024
Desafio de implementar a aplicação  Giropops Senhas em cluster kubernetes e seguro

Objetivo: Criar uma aplicação de gestão de senhas utilizando Docker, Kubernetes, Helm, Kyverno, Cosign, Prometheus, apko e Melange para garantir a segurança, automação, e monitoramento contínuo da aplicação.

O projeto está dividio em pastas 
1 - Docker - Consta a imagem docker e como realizar o build para utilizar uma imagem multistage utilizando chainguard

2 - Kubernetes - Os manifestos kubernetes utilizando a imagem gerada pelo melange/apko.

3 - Melange/Apko - Como gerar a imagem da aplicação utilizando melange e apko . O Pipeline de deploy esta sendo aplicado utilizando github actions e esta localizado em 
