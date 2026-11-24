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

A primeira chamada pega o ambiente "frio" (cold start). Antes de rodar o handler, o Lambda precisa passar pela fase de Init: criar o ambiente de execução (aqui, o contêiner), iniciar o runtime do Python, carregar o código e executar tudo o que está fora do handler, como os `import` e o `contador = 0`. Nas chamadas seguintes esse ambiente já está pronto e é reaproveitado, então só o handler roda.

Nos meus testes isso apareceu claramente no `start-api`: a primeira chamada mostrou `Init Duration` e durou 81,82 ms; a segunda já não teve Init e durou 2,03 ms.

Já no `sam local invoke` as minhas duas invocações demoraram parecido (cerca de 10 s de relógio cada). Percebi que o `invoke` cria um contêiner novo a cada execução, então ali as duas foram cold start.

### 2. O que o campo invocacaoNesteAmbiente mostrou com e sem --warm-containers? O que isso diz sobre guardar estado numa variável global da função?

Nos dois casos o meu contador foi 1, 2, 3. Isso me surpreendeu, porque era diferente do que o roteiro esperava. Olhando o registro, entendi o motivo: na SAM CLI 1.174, o `start-api` já cria o contêiner na subida e o mantém ativo entre as chamadas (o registro mostra "already running"). Com `--warm-containers LAZY` a diferença foi só o momento da criação: o contêiner nasce na primeira requisição (a primeira chamada levou 4,7 s, as outras cerca de 1 s).

Quando o ambiente foi descartado, o contador voltou para 1. Depois do erro de tempo limite e de reiniciar o servidor, a chamada seguinte respondeu `"invocacaoNesteAmbiente": 1`.

Conclusão: a variável global só "lembra" alguma coisa enquanto o mesmo ambiente é reaproveitado, e isso não é garantido. O provedor pode descartar o ambiente a qualquer momento ou criar vários em paralelo, cada um com seu próprio contador. Estado que precisa persistir tem que ficar fora da função: banco de dados, cache ou fila.

### 3. O que aconteceu com demorar = true? Qual é o tempo máximo de uma execução no Lambda, e o que fazer com um trabalho mais longo?

Quando mandei `demorar: true` com `Timeout: 3`, a função foi interrompida no meio do `sleep(5)`: o registro mostrou `timed out after 3 seconds`, a cobrança foi de 3000 ms e o cliente recebeu 502 (`Internal server error`). Depois de mudar para `Timeout: 10` e compilar de novo, a mesma chamada voltou 201, com duração de 5087 ms.

O tempo máximo de uma execução no Lambda é de 15 minutos (900 segundos). Para um trabalho mais longo, a saída é não fazer tudo numa execução só: quebrar o trabalho em etapas menores coordenadas por um orquestrador (como o Step Functions), usar uma fila para distribuir as partes, ou levar o processamento para um serviço sem esse limite, como um contêiner (ECS/Fargate) ou o AWS Batch.

### 4. Quanto custa um milhão de execuções desta função com 128 MB e 100 ms? (us-east-1: US$ 0,20 por milhão de requisições e US$ 0,0000166667 por GB-s)

| Passo | Conta | Resultado |
|---|---|---|
| Memória em GB | 128 MB / 1024 | 0,125 GB |
| Duração em segundos | 100 ms / 1000 | 0,1 s |
| GB-s por execução | 0,125 × 0,1 | 0,0125 GB-s |
| GB-s em 1 milhão de execuções | 0,0125 × 1.000.000 | 12.500 GB-s |
| Custo de computação | 12.500 × 0,0000166667 | US$ 0,2083 |
| Custo das requisições | 1.000.000 × (0,20 / 1.000.000) | US$ 0,20 |
| **Total** | 0,2083 + 0,20 | **≈ US$ 0,41** |

Esse valor desconsidera a faixa sempre gratuita (1 milhão de requisições e 400 mil GB-s por mês). Com ela, esse volume sairia de graça.

### 5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?

**Faz sentido** em tarefas curtas, sem estado e disparadas por evento: gerar a guia de transporte quando uma requisição é aprovada (exatamente esta função), validar um pedido, enviar notificação ao hospital quando a bolsa sai ou chega, gerar um PDF ou um relatório pontual. Essas tarefas acontecem de forma irregular, então pagar só pelo que roda é vantajoso.

**Não faria sentido** no que precisa manter estado ou conexão aberta, ou não tolera atraso: o acompanhamento em tempo real do transporte (posição do veículo, conexão contínua com o app), processamentos longos como planejamento de rotas para muitos pedidos, e o controle de estoque das bolsas se ele fosse guardado na memória da função. Nesses casos, o cold start, o limite de 15 minutos e a falta de estado garantido atrapalham, e um serviço sempre ativo com banco de dados é mais adequado.

## Evidências

As imagens da pasta [evidencias/](evidencias/) são quadros extraídos do vídeo da execução:

| Arquivo | O que mostra |
|---|---|
| `01-template-timeout-3.png` | `template.yaml` com a rota `/guias`, `post` e `Timeout: 3` |
| `02-invoke-evento-201.png` | `sam local invoke` com o evento: status 201 e guia `G-R-001` |
| `03-start-api-3-curls-contador.png` | `start-api` e três `curl`: contador 1, 2, 3; `Init Duration` só na primeira |
| `04-timeout-502.png` | `demorar: true` com Timeout 3: `timed out after 3 seconds` e 502 |
| `05-template-timeout-10.png` | `template.yaml` com `Timeout: 10` |
| `06-timeout-10-sucesso-201.png` | Mesma chamada com Timeout 10: 201 em 5087 ms |

Vídeo (até 3 min): https://luvaplay.com.br/game/Serverless.mp4

## Uso de IA

- **Ferramenta:** Claude Code (Anthropic), dentro do VS Code.
- **Finalidade:** instalar o SAM CLI, gerar o projeto com `sam init`, aplicar as mudanças do Passo 2, diagnosticar o erro do body em base64 e rodar os experimentos para conferir se o roteiro funcionava nesta versão do SAM. Também ajudou a organizar este README, a redigir as respostas a partir dos resultados que obtive e a extrair do vídeo os quadros da pasta `evidencias/`.
- **Como validei:** rodei os comandos e gravei a execução; conferi as saídas (status HTTP, `invocacaoNesteAmbiente`, linhas `REPORT` e `timed out` no registro do SAM). Li e entendi a correção do base64 antes de aceitar. Conferi cada resposta com os números do meu vídeo e refiz a conta de custo.
