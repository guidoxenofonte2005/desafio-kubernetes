## Etapa 00: Setup inicial:
### Setup do Minikube
![Setup do Minikube para clusters locais](images/step_00/minikube_setup.png)
### Validação do Setup
![Validação do setup com comando kubectl cluster-info](images/step_00/kubectl_validate.png)
---

## Etapa 01: Namespace e Primeiro Contato
> Crie um Namespace próprio para o desafio e suba um Pod avulso de teste (pode ser qualquer imagem simples). Inspecione o Pod: seus detalhes, eventos e logs. Depois, delete-o.
### Manifesto criado:
![Manifesto yaml com criação de um namespace chamado 'desafio'](images/step_01/yaml.png)
### Criação do Namespace e Pod:
![alt a](images/step_01/create_namespace_pod.png)
### Inspeção do Pod:
![alt text](images/step_01/get_pod.png)
![alt text](images/step_01/describe_pod.png)
![alt text](images/step_01/logs_pod.png)
### Deleção e Análise
![alt text](images/step_01/delete.png)
Ao deletar um Pod avulso, ele não volta automaticamente, pois não há nenhum provedor/gerenciador para analisar se a quantidade de Pods mínima está suprida.