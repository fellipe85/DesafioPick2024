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

Para a nossa aplicação , criamos uma regra para que sempre tenha que existir um valor de cpu e memoria definido , e também containers como root. 

