# Serverless: guia de transporte do Rota Vital

Atividade individual 1 de Infraestrutura de Software (ADS, CESAR School, 2026.2).

Uma função serverless em Python que recebe uma requisição do Rota Vital e devolve a guia de transporte. Ela é acionada por evento (`sam local invoke`) e por HTTP (`sam local start-api`), e serve para observar cold start, reaproveitamento de ambiente, ausência de estado garantido e limite de tempo de execução.

## Caminho escolhido

**Caminho A: AWS SAM CLI local.** Tudo roda no computador, dentro de contêineres Docker que imitam o ambiente do Lambda. Não precisa de conta AWS.

| Ferramenta | Versão usada |
|---|---|
| AWS SAM CLI | 1.174.0 |
| Docker Desktop | 29.2.1 |
| Runtime da função | Python 3.12 (imagem `public.ecr.aws/lambda/python:3.12`) |
| Sistema | Windows 11 |

## Estrutura

```
template.yaml          # define a função, a rota POST /guias e o Timeout
hello_world/app.py     # handler da função
events/pedido.json     # evento de API Gateway gerado pelo sam local generate-event
evidencias/            # prints dos comandos e das saídas
```

## Como rodar

Pré-requisitos: Docker Desktop aberto (com "Engine running") e SAM CLI instalado (`sam --version`).

No Windows, os comandos com JSON entre aspas simples funcionam como escritos no **Git Bash** ou no **WSL**.

### 1. Build

```bash
sam build --use-container
```

O `--use-container` compila dentro de um contêiner com Python 3.12, então não é preciso ter o Python 3.12 instalado na máquina.

### 2. Invocar com evento

```bash
sam local generate-event apigateway aws-proxy --method POST --path guias \
  --body '{"requisicaoId":"R-001","hospital":"H-Recife","bolsas":["O-NEG-0142"]}' \
  > events/pedido.json
sam local invoke -e events/pedido.json
```

Resposta esperada: `statusCode 201` com `"guia": "G-R-001"`.

### 3. Acionar por HTTP

```bash
sam local start-api
# em outro terminal, três vezes:
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
```

Depois repetir com `sam local start-api --warm-containers LAZY`.

### 4. Limite de tempo

```bash
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-003","demorar":true}'
```

Com `Timeout: 3` a chamada falha. Para o ajuste, trocar `Timeout` para `10` em `Globals` no `template.yaml`, rodar `sam build --use-container` e subir a API de novo. O repositório já está com `Timeout: 10`.

## Ajuste no handler: body em base64

O código do PDF faz `json.loads(event["body"])` direto. Na SAM CLI 1.174, o `sam local generate-event` gera o evento com `"isBase64Encoded": true` e o body codificado em base64, e a primeira invocação falhou com `JSONDecodeError`.

A correção segue o contrato do API Gateway: se `isBase64Encoded` for verdadeiro, decodificar o body antes de ler o JSON. Pelo `start-api` o body chega como texto puro, e o código trata os dois casos.

```python
corpo = event.get("body") or "{}"
if event.get("isBase64Encoded"):
    corpo = base64.b64decode(corpo).decode("utf-8")
pedido = json.loads(corpo)
```

## Resultados observados

### Invocação com evento (`sam local invoke`)

| Invocação | Tempo total | Duration (REPORT) | invocacaoNesteAmbiente |
|---|---|---|---|
| 1ª | 10,01 s | 61,03 ms | 1 |
| 2ª | 9,98 s | 47,41 ms | 1 |

### HTTP com `sam local start-api` (padrão)

| Chamada | Tempo do curl | Duration (REPORT) | Init Duration | invocacaoNesteAmbiente |
|---|---|---|---|---|
| 1ª | 1,13 s | 51,50 ms | 0,04 ms | 1 |
| 2ª | 1,05 s | 2,41 ms | (não aparece) | 2 |
| 3ª | 1,05 s | 1,67 ms | (não aparece) | 3 |

Log do SAM na subida: `Initializing the lambda functions containers` e, a cada chamada, `Lambda function 'HelloWorldFunction' is already running`.

### HTTP com `--warm-containers LAZY`

| Chamada | Tempo do curl | Duration (REPORT) | invocacaoNesteAmbiente |
|---|---|---|---|
| 1ª | 4,73 s | 70,29 ms | 1 |
| 2ª | 1,04 s | 1,46 ms | 2 |
| 3ª | 1,07 s | 1,63 ms | 3 |

### Limite de tempo (`demorar: true`)

| Timeout | Resultado | Log |
|---|---|---|
| 3 s | HTTP 502, `{"message":"Internal server error"}` | `Function 'HelloWorldFunction' timed out after 3 seconds`, Billed Duration 3000 ms |
| 10 s | HTTP 201, `"guia": "G-R-003"` | Duration 5062,20 ms, Billed Duration 5063 ms |

## Respostas

### 1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?

### 2. O que o campo invocacaoNesteAmbiente mostrou com e sem --warm-containers? O que isso diz sobre guardar estado numa variável global da função?

### 3. O que aconteceu com demorar = true? Qual é o tempo máximo de uma execução no Lambda, e o que fazer com um trabalho mais longo?

### 4. Quanto custa um milhão de execuções desta função com 128 MB e 100 ms? (us-east-1: US$ 0,20 por milhão de requisições e US$ 0,0000166667 por GB-s)

### 5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?

## Evidências

Os prints estão na pasta [evidencias/](evidencias/).

Vídeo (até 3 min): https://luvaplay.com.br/game/Serverless.mp4

## Uso de IA

- **Ferramenta:** Claude Code (Anthropic), dentro do VS Code.
- **Finalidade:** instalar o SAM CLI, gerar o projeto com `sam init`, aplicar as mudanças do Passo 2, diagnosticar o erro do body em base64 e rodar os experimentos para conferir se o roteiro funcionava nesta versão do SAM. Também ajudou a organizar este README.
- **Como validei:** rodei os comandos e conferi as saídas (status HTTP, `invocacaoNesteAmbiente`, linhas `REPORT` e `timed out` no log do SAM). Li e entendi a correção do base64 antes de aceitar. As respostas da seção acima foram escritas por mim.
