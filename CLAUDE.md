# Contexto do projeto

Atividade individual 1 (Serverless) de Infraestrutura de Software, CESAR School 2026.2. Entrega: 24/11/2026 às 19h15, link do repositório e do vídeo (até 3 min) no Classroom.

## Estado

- Caminho A (SAM CLI local). SAM 1.174.0 instalado via winget, Docker Desktop 29.2.1.
- Passos 2 a 5 do PDF executados e validados; dados medidos estão no README.
- `sam build --use-container` (máquina só tem Python 3.14).
- Handler decodifica body base64 (`isBase64Encoded`), necessário porque o `generate-event` do SAM 1.174 codifica o body.
- No SAM 1.174 o `start-api` padrão já mantém o contêiner vivo (contador 1, 2, 3), diferente do que o PDF supõe. LAZY só adia a criação para a 1ª chamada.
- Template final com `Timeout: 10`.

## Pendente (Renato)

- Escrever as 5 respostas do README com as próprias palavras.
- Prints em `evidencias/` e vídeo; colar o link do vídeo no README.

## Ambiente no Windows

- Chamar `sam` pelo PowerShell ou Git Bash com o PATH atualizado; no Git Bash, `sam.cmd` com JSON entre aspas pode quebrar ao passar pelo cmd.
- Ao parar o `start-api`, conferir `docker ps`: pode sobrar contêiner `lambda/python:3.12-rapid`.
