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
