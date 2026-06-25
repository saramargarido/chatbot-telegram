# 🌤️ Chatbot de Clima via Telegram

Chatbot que recebe o nome de uma cidade via Telegram, consulta a API OpenWeather e retorna a temperatura atual com uma mensagem humorada gerada pelo Google Gemini.

## Fluxo do Workflow

```
Telegram Trigger → Set (normaliza cidade) → HTTP Request (OpenWeather)
→ Code (Normalize) → IF → [sucesso] Basic LLM Chain (Gemini) → Code (Fallback) → Telegram
                        → [erro] Edit Fields → Telegram
```

### Nós principais

| Nó | Tipo | Função |
|----|------|--------|
| Telegram Trigger | Trigger | Recebe mensagem do usuário |
| Edit Fields1 | Set | Normaliza texto (minúsculas, sem acentos) |
| HTTP Request | HTTP Request | Consulta OpenWeather API |
| Normalize | Code | Trata erros e extrai dados do JSON |
| If | IF | Roteia sucesso/erro |
| Basic LLM Chain | LLM Chain | Gera mensagem humorada via Gemini |
| Fallback | Code | Gera mensagem determinística se Gemini falhar |
| Send a text message | Telegram | Envia resposta ao usuário |

### Nó Gemini (opcional)

O nó **Basic LLM Chain** usa o **Google Gemini** para gerar mensagens criativas e bem-humoradas sobre o clima. Está posicionado após o IF (saída verdadeira), antes do nó Fallback.

- Modelo: `gemini-2.5-flash-lite`
- Temperatura: `0.1`
- On Error: `Continue` — se falhar, o nó Fallback assume automaticamente

Para desativar o Gemini: remova a credencial do nó **Google Gemini Chat Model**. O Fallback assume sem quebrar o fluxo.

---

## Pré-requisitos

- [Docker](https://www.docker.com/) e Docker Compose
- Conta no [Telegram](https://telegram.org/) e um bot criado via [@BotFather](https://t.me/BotFather)
- Chave de API da [OpenWeather](https://openweathermap.org/api)
- (Opcional) Chave de API do [Google AI Studio](https://aistudio.google.com/) para o Gemini
- URL pública HTTPS para webhooks (ngrok, Cloudflare Tunnel, localtunnel, etc.)

---

## Variáveis de ambiente necessárias

| Variável | Descrição |
|----------|-----------|
| `TELEGRAM_BOT_TOKEN` | Token do bot gerado pelo @BotFather |
| `OPENWEATHER_API_KEY` | Chave da API OpenWeather |
| `WEBHOOK_URL` | URL pública HTTPS do seu túnel |
| `NGROK_AUTHTOKEN` | (Opcional) Token do ngrok se usar ngrok como túnel |

> Nenhuma chave real deve ser commitada no repositório. Configure as credenciais diretamente no n8n conforme as instruções abaixo.

---

## Como executar

### 1. Subir o ambiente

```bash
# Clone o repositório
git clone <url-do-repositorio>
cd workflow-chatbot-telegram

# Configure a URL do túnel no docker-compose.yml
# Substitua YOUR_TUNNEL_URL pela sua URL pública HTTPS

# Suba os serviços
docker compose up -d
```

### 2. Expor o n8n via HTTPS

O n8n precisa de uma URL pública HTTPS para receber webhooks do Telegram.

**Opção A — Cloudflare Tunnel:**
```bash
brew install cloudflare/cloudflare/cloudflared
cloudflared tunnel --url http://localhost:5678
```

**Opção B — ngrok:**
```bash
ngrok http 5678
```

Copie a URL gerada e atualize `WEBHOOK_URL` no `docker-compose.yml`, depois reinicie:
```bash
docker compose restart n8n-editor n8n-worker
```

### 3. Importar o workflow

1. Acesse o n8n em `http://localhost:5678`
2. Menu lateral → **Workflows** → **Import from file**
3. Selecione `workflow-chatbot-telegram.json`

### 4. Configurar credenciais no n8n

#### Telegram
1. n8n → **Credentials** → **New** → **Telegram**
2. Cole o token gerado pelo @BotFather (`TELEGRAM_BOT_TOKEN`)
3. Salve e associe ao nó **Telegram Trigger** e **Send a text message**

#### OpenWeather
1. No nó **HTTP Request**, localize o parâmetro `appid`
2. Substitua `YOUR_OPENWEATHER_API_KEY` pela sua chave (`OPENWEATHER_API_KEY`)

#### Google Gemini (opcional)
1. n8n → **Credentials** → **New** → **Google Gemini(PaLM) Api**
2. Cole a chave obtida em [aistudio.google.com](https://aistudio.google.com/apikey)
3. Associe ao nó **Google Gemini Chat Model**

### 5. Ativar o workflow

1. Abra o workflow importado
2. Clique no toggle **Activate** (canto superior direito)

---

## Como usar o chatbot

Envie o nome de uma cidade para o bot no Telegram:

```
São Paulo
London
Tokyo
Rio de Janeiro,RJ,BR
```

### Retorno esperado (sucesso)

```
⛅ *São Paulo, BR*
Nublado

🌡️ 22°C • sensação 23°C
💧 Umidade: 78%
💨 Vento: 14 km/h

Hoje em SP está tão nublado que até o sol pediu home office. ☁️
```

### Retorno esperado (erro)

```
❌ Cidade não encontrada.

Use o formato: Cidade,UF,BR
Ex: São Paulo,SP,BR
```

---

## Testes recomendados

- `London` — cidade simples em inglês
- `São Paulo,SP,BR` — cidade com formato completo
- `Tokyo,JP` — cidade internacional
- `cidadeinexistente123` — deve retornar mensagem de erro

---

## Versões utilizadas

- n8n: `1.122.4`
- Node.js: conforme imagem n8nio/n8n:1.122.4
- PostgreSQL: ankane/pgvector (latest)
- Redis: alpine
