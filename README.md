# Atividade Individual 1 · Serverless com AWS SAM

Aplicação serverless desenvolvida em Python 3.12 utilizando o **AWS Serverless Application Model (AWS SAM)** e **Docker** para emulação local de uma função AWS Lambda integrada ao API Gateway (rota `POST /guias`).

## Estrutura do Projeto

- `hello_world/app.py` — Código-fonte da função Lambda responsável pela emissão de guias hospitalares e controle de invocação por ambiente.
- `template.yaml` — Definição da infraestrutura como código (rota `/guias`, método `POST`, runtime `python3.12` e configurações globais de `Timeout`).
- `events/pedido.json` — Payload de evento simulando uma requisição do Amazon API Gateway (`aws-proxy`).
- `evidencias/` — Capturas de tela dos comandos executados e suas respectivas saídas no terminal.

---

## Caminho Escolhido

Optou-se pela **execução 100% local com AWS SAM CLI + Docker Engine** em ambiente Linux (Debian), utilizando o template *Hello World Example* com runtime **Python 3.12** e empacotamento **Zip**. O build foi realizado dentro da imagem oficial de contêiner da AWS (`--use-container`) para garantir paridade com o ambiente de execução da nuvem sem necessidade de provisionamento remoto.

---

## Como Rodar

### Pré-requisitos
- [Docker Engine](https://docs.docker.com/engine/install/) ativo e em execução
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) instalado

### 1. Compilar a aplicação
Na raiz do projeto, execute:

    sam build --use-container

### 2. Invocar a função diretamente com evento (Passo 3)

    sam local invoke -e events/pedido.json

### 3. Subir a API HTTP localmente (Passo 4)
Modo padrão (novo contêiner a cada requisição):

    sam local start-api

Modo com reaproveitamento de ambiente (*warm containers*):

    sam local start-api --warm-containers LAZY

Em outro terminal, envie requisições `POST` para o endpoint `/guias`:

    curl -X POST http://127.0.0.1:3000/guias \
      -H "Content-Type: application/json" \
      -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'

### 4. Testar o limite de tempo / Timeout (Passo 5)
Com a API em execução, envie uma requisição forçando atraso de 5 segundos:

    curl -X POST http://127.0.0.1:3000/guias \
      -H "Content-Type: application/json" \
      -d '{"requisicaoId":"R-003","demorar":true}'

---

## Evidências

Os prints completos dos comandos e saídas de cada etapa estão disponíveis na pasta [`evidencias/`](./evidencias).

--- 
# Respostas

**1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?**  
A primeira demorou mais porque o ambiente teve que ser criado do zero (cold start), enquanto na segunda o Docker já tinha deixado tudo meio pronto no cache. Na fase de **Init**, a AWS prepara o contêiner, carrega o Python e roda tudo que está fora da função principal (`lambda_handler`), como os `import` e a criação da variável `contador = 0`, antes de começar a atender o pedido.

**2. O que o campo `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`? O que isso diz sobre guardar estado numa variável global da função?**  
Sem o `--warm-containers`, o campo mostrou `1` em todas as chamadas, porque o SAM cria e apaga o contêiner toda vez que recebe um pedido. Com o `--warm-containers LAZY`, o número foi subindo (`1`, `2` e `3`), porque o mesmo contêiner ficou aberto e foi reaproveitado. Isso mostra que a gente não pode usar variáveis globais para salvar dados importantes, pois o contêiner pode ser apagado a qualquer hora (ou a AWS pode abrir vários ao mesmo tempo) e o valor se perde. Elas servem só para coisas temporárias, como cache.

**3. O que aconteceu com `demorar = true`? Qual é o tempo máximo de uma execução no Lambda, e o que fazer com um trabalho mais longo?**  
Com `demorar = true`, o código esperou 5 segundos e passou do limite de 3 segundos que estava no `template.yaml`, então a AWS cortou a execução e deu erro de tempo esgotado (*timeout*). O tempo máximo que uma função Lambda pode rodar na AWS é de **15 minutos**. Se uma tarefa demorar mais que isso, o jeito é dividir o trabalho em partes menores (usando filas como o SQS ou o Step Functions) ou rodar esse código em outro serviço que não tenha esse limite curto, como um contêiner no ECS/Fargate ou um servidor EC2.

**4. Com os preços de `us-east-1` da aula (US$ 0,20 por milhão de requisições e US$ 0,0000166667 por GB-s), quanto custa um milhão de execuções desta função com 128 MB e 100 ms? Mostrem a conta.**  
O custo total fica em **US$ 0,41** (cerca de 41 centavos de dólar):
- **Preço pelas chamadas:** 1 milhão de chamadas = **US$ 0,20**
- **Memória em GB:** $128\text{ MB} / 1024 = 0,125\text{ GB}$
- **Tempo em segundos:** $100\text{ ms} = 0,1\text{ s}$
- **Uso total (GB-s):** $1.000.000 \times 0,125 \times 0,1 = 12.500\text{ GB-s}$
- **Preço pelo processamento:** $12.500 \times 0,0000166667 = \text{US\$ } 0,2083$
- **Soma final:** $\text{US\$ } 0,20 + \text{US\$ } 0,2083 = \mathbf{\text{US\$ } 0,4083}$ (arredondando, **US$ 0,41**).

**5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?**  
- **Onde faz sentido:** Em ações rápidas que acontecem só de vez em quando, como gerar uma guia de pedido de sangue (`POST /guias`), validar um cadastro ou mandar um alerta rápido para o hospital quando uma bolsa sair para entrega.  
- **Onde não faz sentido:** No rastreamento ao vivo do GPS das ambulâncias/veículos, porque o envio de localização acontece sem parar o tempo todo (o que sairia caro e precisaria de conexão aberta), nem em cálculos muito pesados de rotas que demorem mais de 15 minutos para terminar.
