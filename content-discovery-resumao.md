# 🔍 Resumão Content Discovery — Reconhecimento Web

> Anotações de estudo (TryHackMe + complementos) para o repositório de Cybersecurity.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [O que é Content Discovery](#1-o-que-é-content-discovery)
2. [Manual — Arquivos comuns](#2-manual--arquivos-comuns)
3. [Manual — Headers HTTP e stack do framework](#3-manual--headers-http-e-stack-do-framework)
4. [OSINT — Motores de busca e ferramentas web](#4-osint--motores-de-busca-e-ferramentas-web)
5. [OSINT — Repositórios e arquivos históricos](#5-osint--repositórios-e-arquivos-históricos)
6. [Automatizado — Gobuster: fundamentos e modo `dir`](#6-automatizado--gobuster-fundamentos-e-modo-dir)
7. [Automatizado — Gobuster: subdomínios (`dns`) e virtual hosts (`vhost`)](#7-automatizado--gobuster-subdomínios-dns-e-virtual-hosts-vhost)
8. [Workflow recomendado](#8-workflow-recomendado)
9. [Cheat sheet final](#9-cheat-sheet-final)
10. [Erros comuns](#10-erros-comuns)

---

## 1. O que é Content Discovery

**Conteúdo** = qualquer coisa hospedada num servidor web: páginas, arquivos, diretórios, painéis admin, configs, backups. **Content discovery** é o processo de achar conteúdo que não foi pensado pra ser público ou que não está linkado em lugar nenhum óbvio.

### As três abordagens

| Método | Ideia central |
|---|---|
| **Manual** | Checar arquivos de convenção, headers, código-fonte — rápido, acha vazamentos óbvios |
| **OSINT** | Usar fontes externas que já indexaram/arquivaram informação do alvo |
| **Automatizado** | Testar milhares de caminhos possíveis via wordlist, em escala |

> **Regra de ouro:** rodar os três métodos antes de partir pra exploração. Nenhum cobre tudo sozinho — manual acha o que foi esquecido, OSINT acha o que vazou em outro lugar, automatizado cobre o volume que nenhum dos dois cobre a mão.

---

## 2. Manual — Arquivos comuns

### `robots.txt`

Instrui crawlers sobre o que podem/não podem indexar.

```
User-agent: *
Allow: /
Disallow: /staff-portal
```

📌 É só uma **diretriz educada** pra bots — não é controle de acesso. Nada impede acessar `/staff-portal` direto na URL. Na prática, funciona como uma **lista pronta de lugares interessantes**, já que o dono do site está, sem querer, indicando o que quer esconder.

### `sitemap.xml`

Função oposta: lista as páginas que o dono **quer** indexadas (ajuda SEO). Pode revelar páginas de staging, conteúdo antigo, ou URLs difíceis de achar navegando normalmente.

```xml
<url>
  <loc>http://alvo.com/news/article?id=1</loc>
</url>
```

📌 Parâmetros como `?id=` já são pista pra testes futuros de **IDOR** ou **SQL injection**.

---

## 3. Manual — Headers HTTP e stack do framework

### Fingerprint via headers

```bash
curl http://alvo.com -v
```

```
Server: nginx/1.18.0 (Ubuntu)
X-Powered-By: THM-Framework
```

| Header | Revela |
|---|---|
| `Server` | Software do servidor web e versão |
| `X-Powered-By` | Framework/linguagem da aplicação |

📌 Versão exata → buscar CVEs conhecidas (`searchsploit "nome da versão"`).

### Comentários HTML

Páginas geradas por frameworks/CMS costumam deixar comentários reveladores no código-fonte:

```html
<!-- Page Generated using FRAMEWORK v1.2 (link oficial) -->
```

### Fluxo: identificar → documentação → credenciais padrão

```text
1) Achar o framework (header, comentário, favicon)
2) Visitar a documentação oficial do framework
3) A doc geralmente revela: estrutura de diretórios padrão, caminho do painel admin
4) Aplicar esse caminho no alvo
5) Testar credenciais padrão (admin/admin e afins — muita gente nunca troca)
```

---

## 4. OSINT — Motores de busca e ferramentas web

### Google Hacking / Dorking

| Filtro | Exemplo | Função |
|---|---|---|
| `site` | `site:alvo.com` | Só resultados daquele domínio |
| `inurl` | `inurl:admin` | Palavra na URL |
| `filetype` | `filetype:pdf` | Resultados de um tipo de arquivo |
| `intitle` | `intitle:admin` | Palavra no título da página |
| `intext` | `intext:password` | Palavra no corpo do texto |
| `cache` | `cache:alvo.com` | Versão em cache do Google |

Combinando:
```
site:alvo.com filetype:pdf
```

📌 O Google indexa tudo que encontra e não foi bloqueado — inclusive login forgotten, docs internos vazados por engano.

### Wappalyzer

Extensão/ferramenta que identifica instantaneamente toda a stack de um site: framework, CMS, CDN, analytics, pagamento — e, quando possível, a **versão exata**. Automatiza o que daria pra fazer manualmente lendo headers/comentários.

---

## 5. OSINT — Repositórios e arquivos históricos

### Wayback Machine (web.archive.org)

Arquivo da internet com snapshots desde os anos 90. Acha páginas **já removidas** do site atual: formulários antigos, endpoints esquecidos, conteúdo publicado e retirado rapidamente.

📌 Remover um link **não** remove o endpoint — se o backend nunca desativou a rota, ela pode continuar ativa mesmo sem estar mais visível.

### GitHub

Buscar nome da empresa/domínio no GitHub pode achar repositórios com API keys, credenciais, `.env` vazados.

```text
1) Achar o repositório do alvo
2) NÃO olhar só os arquivos atuais — olhar o HISTÓRICO de commits
3) git log -p  → mostra o diff de cada commit
4) Procurar commits tipo "remove API key" → aponta pro commit ANTERIOR com o segredo
5) git show <hash>  → ver o conteúdo daquele commit
```

📌 **Remover um arquivo num commit novo não apaga ele do Git** — o histórico de versões é preservado por padrão, mesmo quando o arquivo "sumiu" da versão atual.

### S3 Buckets

```
https://{nome}.s3.amazonaws.com
```

Padrões comuns de nomenclatura pra testar:
```
{empresa}-assets
{empresa}-backup
{empresa}-www
{empresa}-dev
```

Também vale procurar URLs de bucket já referenciadas no page source ou em repositórios GitHub do alvo.

---

## 6. Automatizado — Gobuster: fundamentos e modo `dir`

Pré-instalado no AttackBox/Kali. Três modos: `dir` (diretórios/arquivos), `dns` (subdomínios), `vhost` (virtual hosts).

### Flags globais

| Flag | Função |
|---|---|
| `-t` / `--threads` | Threads simultâneas (padrão: 10) |
| `-w` / `--wordlist` | Wordlist (obrigatório em todos os modos) |
| `-o` / `--output` | Salva resultado em arquivo |
| `--delay` | Espera entre requisições (contra rate limiting) |

### Wordlists (SecLists)

```bash
# Instalar no Kali (se não vier por padrão):
sudo apt install seclists
# Instala em /usr/share/seclists/ (minúsculo)

# No AttackBox o path costuma ser:
# /usr/share/wordlists/SecLists/
```

Arquivos mais usados:
```
Discovery/Web-Content/common.txt
Discovery/Web-Content/directory-list-2.3-medium.txt
```

### Modo `dir`

```bash
gobuster dir -u "http://alvo.com" -w /path/to/wordlist
```

| Flag | Função |
|---|---|
| `-x` / `--extensions` | Extensões a testar (`-x .php,.txt,.js`) |
| `-r` / `--followredirect` | Segue redirecionamentos |
| `-k` / `--no-tls-validation` | Pula verificação TLS (labs) |
| `-s` / `--status-codes` | Só mostra status específicos (`-s 200,301`) |

Exemplo de output interessante:
```
/customers            (Status: 302) [--> /customers/login]   ← área autenticada
/development.log      (Status: 200) [Size: 27]               ← log pode vazar info
```

---

## 7. Automatizado — Gobuster: subdomínios (`dns`) e virtual hosts (`vhost`)

### Conceito-chave: subdomínio ≠ virtual host

| Conceito | Resolvido por | Como funciona |
|---|---|---|
| **Subdomínio** | **DNS** | `blog.alvo.com` é um registro DNS apontando pra um IP |
| **Virtual Host (vhost)** | **Servidor web** | Vários sites no mesmo IP; o servidor decide qual servir pelo header `Host:` |

> Analogia: subdomínio é "pergunta pra central telefônica" (DNS) — você nunca bate numa porta, só confirma se o nome existe na lista. Vhost é "bater sempre na mesma porta, mudando o nome que você fala" — o servidor reage diferente dependendo do `Host:` que você manda, mesmo sem registro DNS público nenhum.

### Modo `dns`

```bash
gobuster dns --domain alvo.com -w <wordlist> --wildcard --resolver <IP_DO_ALVO>
```

| Flag | Função |
|---|---|
| `--domain` (`--do`) | Domínio alvo (**atenção:** em versões recentes, `-d` virou atalho de `--delay`, não de `--domain`!) |
| `--wildcard` | Força enumeração mesmo com DNS wildcard detectado |
| `-i` / `--show-ips` | Mostra o IP resolvido |
| `--resolver` | Usa um servidor DNS customizado — essencial em lab isolado, bypassa precisar editar `/etc/resolv.conf` do sistema |

### Modo `vhost`

```bash
gobuster vhost -u "http://<IP>" --domain alvo.com -w <wordlist> --append-domain --exclude-length <tamanho>
```

| Flag | Função |
|---|---|
| `--append-domain` | Combina cada palavra da wordlist com o domínio |
| `--exclude-length` | Filtra respostas de tamanho igual (geralmente a página padrão, retornada pra qualquer `Host:` desconhecido) |

📌 Resultado "nenhum vhost encontrado" é uma resposta **válida** — só indica que o servidor não esconde nada configurado fora do que já é público via DNS.

---

## 8. Workflow recomendado

```text
1) Manual rápido
   curl -v <alvo>                  → headers
   /robots.txt, /sitemap.xml       → arquivos de convenção
   view-source                     → comentários, links escondidos

2) Identificar a stack
   Headers (X-Powered-By) + Wappalyzer + comentários no HTML
   → buscar documentação oficial do framework/CMS

3) OSINT
   site:alvo.com (Google dorking)
   Wayback Machine → endpoints antigos
   GitHub → nome da empresa, histórico de commits
   S3 → padrões de nome de bucket

4) Automatizado
   gobuster dir  -u <alvo> -w <wordlist>              → diretórios/arquivos
   gobuster dns  --domain <alvo> -w <wordlist>         → subdomínios
   gobuster vhost -u <IP> --domain <alvo> -w <wordlist> → vhosts escondidos

5) Consolidar achados → alimentam a próxima fase (exploitation)
```

---

## 9. Cheat sheet final

```bash
# Manual
curl <alvo> -v
curl <alvo>/robots.txt
curl <alvo>/sitemap.xml

# Gobuster dir
gobuster dir -u <alvo> -w /usr/share/seclists/Discovery/Web-Content/common.txt
gobuster dir -u <alvo> -w <wordlist> -x .php,.txt,.js -s 200,301

# Gobuster dns
gobuster dns --domain <dominio> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --wildcard --resolver <IP>

# Gobuster vhost
gobuster vhost -u "http://<IP>" --domain <dominio> -w <wordlist> --append-domain --exclude-length <tamanho>

# Google dorking
site:<dominio> filetype:pdf
site:<dominio> inurl:admin
site:<dominio> intext:password
```

---

## 10. Erros comuns

- ❌ Achar que `robots.txt` é controle de acesso — é só uma sugestão pra bots.
- ❌ Ignorar o histórico de commits do Git — segredo "removido" ainda existe no histórico.
- ❌ Confundir subdomínio com virtual host — mecanismos e ferramentas diferentes (DNS vs. header `Host:`).
- ❌ Usar `-d` esperando que seja `--domain` em versões recentes do Gobuster (virou `--delay`).
- ❌ Usar o path de wordlist do AttackBox (`/usr/share/wordlists/SecLists/`) achando que é igual no Kali próprio (`/usr/share/seclists/`).
- ❌ Rodar `gobuster dns` sem `--resolver` num lab isolado — a resolução de DNS do sistema não vai funcionar sem configuração extra.
- ❌ Confiar só num método (manual OU OSINT OU automatizado) — cada um cobre uma lacuna que os outros não cobrem.

---

## 📚 Referências

- [TryHackMe – Content Discovery Room](https://tryhackme.com/room/contentdiscovery)
- [SecLists (GitHub)](https://github.com/danielmiessler/SecLists)
- [Gobuster (GitHub)](https://github.com/OJ/gobuster)
- [Wayback Machine](https://web.archive.org/)
- [Wikipedia: Google Hacking](https://en.wikipedia.org/wiki/Google_hacking)
