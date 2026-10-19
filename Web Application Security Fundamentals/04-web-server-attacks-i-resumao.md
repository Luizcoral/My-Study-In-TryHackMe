# 🕵️ Resumão: Web Server Attacks I — Misconfigurations (Apache, Python, Node.js, Nginx)

> Anotações de estudo (TryHackMe — Web Application Security Fundamentals) para o repositório de Cybersecurity.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [Ideia central da sala](#1-ideia-central-da-sala)
2. [Identificando o servidor web](#2-identificando-o-servidor-web)
3. [Python HTTP Server](#3-python-http-server)
4. [Apache2](#4-apache2)
5. [Node.js (Express)](#5-nodejs-express)
6. [Nginx](#6-nginx)
7. [Padrões comuns a todos os servidores](#7-padrões-comuns-a-todos-os-servidores)
8. [Tabela-resumo das 4 stacks](#8-tabela-resumo-das-4-stacks)
9. [Checklist de auditoria rápida](#9-checklist-de-auditoria-rápida)
10. [Erros comuns](#10-erros-comuns)

---

## 1. Ideia central da sala

Diferente da sala de CVEs (onde cada stack tinha um **bug de código** específico), aqui o foco é **misconfiguration** — erro de configuração, não vulnerabilidade de software. A sala para na fase de identificação: sem shells, sem RCE, sem escalonamento. O objetivo é reconhecimento e análise do que está exposto e por quê.

> **Configuração padrão prioriza facilidade de instalar, não segurança.** Version disclosure, directory listing e páginas de status vêm ligadas por padrão para fins de diagnóstico do administrador. Desligar isso exige ação deliberada — por isso aparece com tanta frequência em engajamentos reais.

---

## 2. Identificando o servidor web

### Server header

```bash
curl -sI http://MACHINE_IP:PORTA
```

| Porta (lab) | Servidor | Header `Server:` padrão |
|---|---|---|
| 80 | Apache2 | `Apache/2.4.x (Ubuntu)` |
| 8000 | Python HTTP Server | `SimpleHTTP/0.6 Python/3.xx.x` |
| 3000 | Node.js Express | *(nenhum — ver `X-Powered-By`)* |
| 8080 | Nginx | `nginx/1.xx.x` |

### X-Powered-By (quando o Server não ajuda)

Express não manda `Server:` por padrão (nem o módulo `http` do Node manda). O fingerprint confiável aqui é:

```
X-Powered-By: Express
```

### Páginas de erro (quando o header está suprimido)

```bash
curl -s http://MACHINE_IP:PORTA/nonexistent-page-xyz
```

Cada servidor tem uma "cara" de 404 diferente: Python devolve texto puro, Nginx carimba a versão no rodapé do HTML, Apache coloca o nome no corpo da página. Útil quando o header `Server` foi escondido de propósito.

### Via navegador (DevTools)

F12 → aba **Network** → recarregar a página → selecionar a requisição principal (`Type: document`) → ver **Response Headers**. Mesma informação do `curl -I`, só que visual.

---

## 3. Python HTTP Server

```bash
python3 -m http.server 8000
```

Comando de conveniência que vira exposição acidental quando esquecido rodando em produção.

### O que ele serve

- **Tudo** que está na pasta de trabalho, sem exceção
- **Dotfiles** (`.env`, etc.) normalmente escondidos no Linux — servidos normalmente, sem distinção
- Não tem `.htaccess` equivalente, sem blocklist, sem configuração

### Comandos usados

```bash
curl -s http://MACHINE_IP:8000/                  # directory listing (se não tiver index.html)
curl -s http://MACHINE_IP:8000/.env              # dotfiles acessíveis
curl -s http://MACHINE_IP:8000/config.txt
curl -s http://MACHINE_IP:8000/notes.txt
curl -s http://MACHINE_IP:8000/backup.zip -o backup.zip
unzip backup.zip -d backup-contents/
```

**Achado no lab:** `.env` com `SECRET_KEY` e `DATABASE_URL` de produção; `notes.txt` com credenciais de admin em texto puro (`admin/admin123`); `backup.zip` com dump de banco contendo a flag.

🏁 **Flag:** `THM{py_server_exposed}`

> **Lição:** não é exploração — o servidor funciona exatamente como projetado. O erro é 100% de lugar (rodando onde não devia) + conteúdo (pasta com segredos dentro).

---

## 4. Apache2

### Os 4 passos de investigação

```bash
# 1) Versão exata
curl -sI http://MACHINE_IP:80 | grep -i server
# Server: Apache/2.4.58 (Ubuntu)

# 2) Directory listing (Options +Indexes)
curl -s http://MACHINE_IP/files/

# 3) mod_status — deveria ser só localhost
curl -s http://MACHINE_IP/server-status

# 4) Gobuster — acha o que não tem link
gobuster dir -u http://MACHINE_IP:80 -w /usr/share/seclists/Discovery/Web-Content/common.txt -x bak,txt -t 20
```

⚠️ No Kali, o pacote `seclists` instala em `/usr/share/seclists/` (minúsculo) — o caminho `SecLists/` com maiúsculas **não existe**, é case-sensitive.

### `mod_status` em detalhe

Por padrão restrito com `Require local` em `conf-available/security.conf`. Quebra quando um `Require all granted` em outro lugar da config (geralmente num virtual host) sobrescreve essa regra sem que ninguém tenha mexido diretamente no módulo. Vaza: versão, uptime, conexões ativas por estado, e os caminhos sendo requisitados **em tempo real** por outros usuários.

**Achado no lab:** `/files/` → `employees.csv` + `internal-notes.txt` (flag); `/server-status` exposto; `backup.bak` achado via Gobuster com credenciais de banco em comentário.

🏁 **Flag:** `THM{apache_dir_listing}`

---

## 5. Node.js (Express)

Diferente dos outros — não serve arquivos estáticos de uma pasta configurada, roda **código de aplicação** que decide o que devolver a cada requisição. A flexibilidade é onde moram os erros.

### A cadeia de 5 passos

```bash
# 1) Fingerprint
curl -sI http://MACHINE_IP:3000
# X-Powered-By: Express

# 2) Provocar erro — ver o que vaza em modo debug
curl -s http://MACHINE_IP:3000/api/users | python3 -m json.tool
# → stack trace completo: caminhos internos, módulos, query SQL

# 3) Endpoint de debug listando rotas
curl -s http://MACHINE_IP:3000/api/routes

# 4) Variáveis de ambiente expostas
curl -s http://MACHINE_IP:3000/api/debug/env

# 5) Arquivos estáticos (express.static())
curl -s http://MACHINE_IP:3000/static/config.js
```

⚠️ `express.static()` bloqueia dotfiles por padrão (404 silencioso para `.env`) — comportamento oposto ao Python HTTP Server.

⚠️ `/api/routes`, quando existe, normalmente lê `app._router.stack` (propriedade interna do Express) — estrutura instável entre versões; Express 5 quebrou implementações feitas para Express 4.

**Achado no lab:** stack trace com caminho `/opt/nodeapp/app.js` e query `SELECT * FROM users`; rotas enumeradas; `DB_PASSWORD` e `NODE_ENV=development` expostos; `config.js` com `API_BASE` apontando pra API interna (`*.company.local`) + flag.

🏁 **Flag:** `THM{node_debug_exposed}`

---

## 6. Nginx

Estrutura paralela ao Apache, vocabulário diferente. Geralmente atua como reverse proxy/load balancer na frente de outra aplicação.

```bash
# 1) Versão
curl -sI http://MACHINE_IP:8080 | grep -i server
# Server: nginx/1.24.0 (Ubuntu)

# Confirmar se server_tokens está REALMENTE off (também afeta rodapé de erro)
curl -s http://MACHINE_IP:8080/nonexistent-path

# 2) Directory listing — autoindex (não vem ligado por padrão, diferente do Apache)
curl -s http://MACHINE_IP:8080/files/

# 3) stub_status — equivalente ao mod_status do Apache
curl -s http://MACHINE_IP:8080/nginx_status
```

`server_tokens` controla **ao mesmo tempo** o header `Server:` e a versão no rodapé de erro — uma única diretiva, dois lugares.

Saída do `/nginx_status` (3 números da 2ª linha, em ordem): conexões aceitas, conexões tratadas, total de requisições.

**Achado no lab:** `/nginx_status` liberado pra qualquer IP (deveria ser só `127.0.0.1`); `/files/` com `deploy-notes.txt`, `server-config.txt`, `old-backup.tar.gz`.

---

## 7. Padrões comuns a todos os servidores

### Security headers — nenhum vem por padrão, em nenhum dos 4

```bash
for port in 80 8000 3000 8080; do
  echo "=== Port $port ==="
  curl -sI http://MACHINE_IP:$port/ | grep -iE "x-frame-options|x-content-type|content-security-policy|strict-transport|referrer-policy" || echo "(no security headers found)"
done
```

| Header | Protege contra |
|---|---|
| `X-Frame-Options` | Clickjacking (embutir seu site num iframe de outro domínio) |
| `X-Content-Type-Options` | MIME sniffing (navegador "adivinhar" tipo de arquivo errado) |
| `Content-Security-Policy` | Restringe de onde scripts/estilos podem carregar |
| `Referrer-Policy` | Vazamento de URL de origem pra outros sites |
| `Strict-Transport-Security` | Força HTTPS (só relevante em servidores HTTPS) |

📌 `X-Frame-Options` é tecnicamente substituído por `Content-Security-Policy: frame-ancestors` — sites modernos hardenizados podem usar só CSP e omitir o `X-Frame-Options` de propósito. Checar os dois ao reportar um achado.

### Automação com Nikto

```bash
nikto -h http://MACHINE_IP:80 -nointeractive
```

Confirma em segundos o que levaria várias requisições manuais: `/server-status` exposto, `backup.bak`, directory listing em `/files/`, headers de segurança ausentes. Pra um scan mais rápido/filtrado:

```bash
nikto -h TARGET -Tuning 123   # códigos de tuning são concatenados, não separados por vírgula
```

---

## 8. Tabela-resumo das 4 stacks

| | Apache | Python HTTP | Node.js | Nginx |
|---|---|---|---|---|
| Vazamento de versão | Sim | Sim | Parcial (só `X-Powered-By`) | Sim |
| Directory listing | `/files/` | Raiz inteira | N/A | `/files/` |
| Endpoint de status/debug exposto | `/server-status` | N/A | `/api/debug/env`, `/api/routes` | `/nginx_status` |
| Arquivo sensível acessível | `backup.bak`, `internal-notes.txt` | `.env`, `notes.txt`, `backup.zip` | `config.js` | `server-config.txt`, `deploy-notes.txt` |
| Security headers ausentes | Todos | Todos | Todos | Todos |

---

## 9. Checklist de auditoria rápida

```text
1) curl -sI <url>                         → Server, X-Powered-By, cookies
2) curl -s <url>/caminho-inexistente       → rodapé de erro pode vazar versão
3) Testar /files/, /static/, raiz          → directory listing habilitado?
4) Testar /server-status, /nginx_status    → painel de monitoramento vazando?
5) Node/Express: testar /api/routes, /api/debug/env, provocar erro 500
6) gobuster dir -u <url> -w <wordlist> -x bak,txt  → arquivo sem link nenhum
7) for port in <portas>; do curl -sI ... | grep -iE "x-frame-options|..."; done
8) nikto -h <url> -nointeractive          → confirmação automatizada rápida
```

---

## 10. Erros comuns

- ❌ Supor que falta de header `Server:` significa stack "mais segura" — pode só ser comportamento padrão do framework (caso do Express).
- ❌ Confiar só na listagem de diretório — arquivos sem link nenhum (`backup.bak`) só aparecem com Gobuster.
- ❌ Esquecer que no Kali o pacote SecLists instala em `/usr/share/seclists/` minúsculo, não `SecLists/`.
- ❌ Testar `.env` numa rota `express.static()` e concluir que não existe — pode só estar sendo bloqueado pelo middleware (dotfiles dão 404 silencioso no Express, diferente do Python).
- ❌ Parar na confirmação do framework — cada stack tem um **comportamento de exposição próprio** que precisa ser testado individualmente.
- ❌ Ignorar a auditoria de security headers achando que "não é bug de verdade" — é um achado válido e rápido de reportar em qualquer engajamento.

---

## 📚 Referências

- [TryHackMe – Web Application Security Fundamentals](https://tryhackme.com/module/web-application-security-fundamentals)
- [Apache mod_status documentation](https://httpd.apache.org/docs/2.4/mod/mod_status.html)
- [Nginx stub_status module](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [Nikto GitHub](https://github.com/sullo/nikto)
