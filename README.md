# Automação de Backoffice com WebSocket

Antes, a equipe de backoffice coletava os dados do lead e registrava a indicação manualmente no sistema. A automação conecta um fluxo do n8n a uma extensão Chrome para executar a indicação no navegador e devolver o status ao fluxo.

Este repositório contém o servidor Node.js que conecta o n8n à extensão e um cliente de demonstração (`cliente.js`) que simula as mensagens da extensão para testes locais.

## Fluxo da automação

1. O n8n busca os dados do lead e envia um `POST /comando` com o ID da extensão de destino e os dados da indicação.
2. O servidor Express recebe a requisição, localiza a conexão pelo ID e encaminha o comando à extensão por WebSocket.
3. A extensão Chrome executa um script que registra a indicação no sistema pelo navegador.
4. A extensão devolve o resultado ao servidor pela mesma conexão WebSocket.
5. O servidor envia o status da indicação ao webhook do n8n, que recebe o retorno da operação.

## Como funciona o heartbeat

A extensão e o cliente de demonstração enviam `{"tipo":"ping"}` a cada 20 segundos, e o servidor responde com `{"tipo":"pong"}`. Essas são mensagens JSON da aplicação. A troca mantém tráfego periódico na conexão e ajuda a evitar desconexões por inatividade. O cliente de demonstração não monitora a chegada do `pong` nem tenta reconectar automaticamente.

Ao se conectar, cada cliente se identifica com um ID e um nome. O servidor guarda a conexão em memória e remove o registro quando recebe o evento de fechamento.

## Tecnologias

- **Node.js** para executar o cliente e o servidor.
- **Express** para disponibilizar a API HTTP.
- **ws** para criar o servidor e o cliente WebSocket.
- **n8n** para buscar os dados do lead, chamar a API e receber o status por webhook.
- **Extensão Chrome (JavaScript)** para executar a indicação no navegador.
- **JSON** como formato das mensagens HTTP e WebSocket.

## Como testar o servidor localmente

Use Node.js 18 ou superior. Na raiz do projeto, instale as dependências e inicie os dois servidores com o mesmo comando de entrada.

```bash
npm ci
node index.js
```

| Serviço | Endereço local |
| --- | --- |
| WebSocket | `ws://localhost:8080` |
| API HTTP | `http://localhost:3000` |

Em outro terminal, inicie o cliente de exemplo. Ele se registra com o ID `pc1`.

```bash
node cliente.js
```

## Como enviar um comando

Com o cliente conectado, envie uma requisição à API. O exemplo usa sintaxe de Bash ou Git Bash.

```bash
curl -X POST http://localhost:3000/comando \
  -H "Content-Type: application/json" \
  -d '{"id":"pc1","tipo":"indicado"}'
```

O servidor retorna `{"mensagem":"comando enviado com sucesso!"}` quando encaminha o comando. A resposta HTTP confirma o encaminhamento. O status da indicação chega depois, pelo webhook. Se o ID não estiver cadastrado, a API retorna HTTP `404`. Ao receber o tipo `indicado`, o cliente de exemplo simula a execução e responde com uma mensagem do tipo `resultado` e `sucesso: true`.

## Integração com n8n

Configure o fluxo do n8n para enviar os dados do lead e o ID da extensão via `POST /comando`, usando um endereço do servidor acessível ao n8n. Para receber o resultado, substitua a URL em [src/services/n8n.service.js](src/services/n8n.service.js) pela URL de um webhook seu que aceite `POST` e responda com JSON. Se usar a URL de teste do n8n, coloque o webhook em modo de escuta antes de enviar o comando. O servidor encaminha o resultado recebido do cliente no corpo da requisição.

## Estrutura do projeto

| Arquivo ou pasta | Responsabilidade |
| --- | --- |
| `index.js` | Inicia os serviços HTTP e WebSocket. |
| `https.js` | Configura a API HTTP com Express. |
| `server.js` | Processa identificação, heartbeat, resultados e fechamento de conexões. |
| `cliente.js` | Simula um cliente conectado que envia pings e responde a comandos. |
| `src/routes/` e `src/controllers/` | Definem a rota `POST /comando` e encaminham o comando ao cliente pelo ID. |
| `src/services/` | Mantêm o registro de conexões em memória e enviam resultados ao n8n. |
