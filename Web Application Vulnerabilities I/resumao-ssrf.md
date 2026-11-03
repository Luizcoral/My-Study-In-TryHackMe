# 🛡️ Resumão Intro to SSRF — Anotações TryHackMe

> Sala do módulo **Web Application Vulnerabilities I**. Cobre Server-Side Request Forgery de ponta a ponta: conceito, tipos, vetores, detecção, bypass de defesas e exploração prática.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, PortSwigger Academy, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [O que é SSRF](#1-o-que-é-ssrf)
2. [Os 2 tipos](#2-os-2-tipos)
3. [Impacto](#3-impacto)
4. [Os 4 vetores](#4-os-4-vetores)
5. [Como encontrar](#5-como-encontrar)
6. [Confirmando Blind SSRF](#6-confirmando-blind-ssrf)
7. [Bypass de defesas](#7-bypass-de-defesas)
8. [🎯 Lab prático (Acme avatar + traversal)](#8--lab-prático-acme-avatar--traversal)
9. [Mitigação](#9-mitigação)
10. [Erros comuns](#10-erros-comuns)

---

## 1. O que é SSRF

O atacante força o **servidor** a fazer requisições HTTP pra um destino que **ele** escolhe, manipulando um parâmetro que a app usa pra montar a requisição.

**CSRF vs SSRF (não confunda):**

| | CSRF | SSRF |
|---|---|---|
| Quem faz a requisição | Navegador da **vítima** | O **servidor** |
| Abusa de | Confiança do site no navegador | Confiança da **rede interna** no servidor |
| Alvo | Ação na conta da vítima | Serviços internos, metadata da nuvem |

> O **S** é de **Server-Side**. Quem controla pra onde o servidor aponta **herda a confiança** que a rede interna tem nele.

---

## 2. Os 2 tipos

| Tipo | Resposta visível? | Como explorar |
|---|---|---|
| **Regular SSRF** | **Sim** | A resposta da requisição interna volta na página. Lê direto |
| **Blind SSRF** | **Não** | Mensagem de sucesso fixa. Confirma com callback (Collaborator) ou timing |

> Mesmo padrão "vejo / sou cego" do SQLi (In-band/Blind) e XSS (normal/Blind).

---

## 3. Impacto

| Impacto | Exemplo |
|---|---|
| **Endpoints internos** | Painéis admin não expostos; burla controle por IP (requisição vem do servidor) |
| **Dados sensíveis** | Bancos/APIs internas que confiam na posição de rede |
| **Recon interno** | Mapear hosts/portas por tempo, status, erros |
| **Cloud metadata** ⭐ | `169.254.169.254` (AWS/GCP/Azure) → **credenciais temporárias**, IAM |
| **Vazamento de credenciais** | Tokens entre serviços internos em HTTP |

> **`169.254.169.254`** = endpoint de metadata das nuvens. Decore. Chegar nele = roubar credenciais da conta inteira (caso real: Capital One 2019, 100M+ clientes).

---

## 4. Os 4 vetores

**1. URL completa num parâmetro** (mais direto):
```
?server=api  →  https://server.website.thm/api/item?id=2
```

**2. URL parcial (só hostname):**
```
?server=attacker.com  →  requisição vai pro teu servidor
```

**3. Path traversal na URL:**
```
?url=/item/123/details  →  normal
?url=/../admin          →  https://website.thm/admin
```

**4. Campos hidden** (sorrateiro, só vendo o source/Burp):
```html
<input type="hidden" name="avatar" value="/images/avatars/default.png">
```

**Exemplo do exercício (neutralizar o sufixo com `&x=`):**
```
server=server.website.thm/flag?id=9&x=
→ https://server.website.thm/flag?id=9&x=.website.thm/api/item?id=2
                                      └── tudo isso vira valor de "x", ignorado ──┘
```

---

## 5. Como encontrar

Features que costumam ter SSRF:

| Feature | Por quê |
|---|---|
| **Webhook** | App faz requisição à URL que você deu pra "verificar" |
| **Geração de PDF/relatório** | Servidor busca conteúdo de uma URL pra renderizar |
| **Preview/unfurling de link** | Pega título/thumbnail de um link |
| **Import de arquivo por URL** | Servidor baixa de local remoto |
| **Settings de integração** | URLs de terceiros consultadas pelo servidor |

> Regra: todo `request`/`fetch`/`download` numa URL **vinda do usuário** é candidato.

---

## 6. Confirmando Blind SSRF

| Método | Como funciona |
|---|---|
| **HTTP logger** (requestbin) | Dá a URL do logger como payload; checa o dashboard |
| **Burp Collaborator** | Domínio único que loga callbacks HTTP **e DNS** |
| **Listener próprio** (`python3 -m http.server`) | Observa conexões chegando |
| **Timing analysis** | Host interno que existe vs não existe → diferença de tempo |
| **Error-based** | Erros diferentes pra host alcançável vs não |

> **DNS callback** é chave: o firewall às vezes bloqueia HTTP de saída mas **permite DNS**. Você confirma só pela consulta DNS.

---

## 7. Bypass de defesas

### Deny list (bloqueia alvos conhecidos) — frágil

`127.0.0.1` tem mil formas:

| Representação | Valor |
|---|---|
| Padrão | `127.0.0.1` |
| Decimal | `2130706433` |
| Octal | `017700000001` |
| Abreviado | `127.1`, `0`, `0.0.0.0` |
| Wildcard | `127.*.*.*` |
| IPv6 | `[::1]` |
| DNS | `127.0.0.1.nip.io` |

> **nip.io**: `127.0.0.1.nip.io` **resolve pra** `127.0.0.1`, mas o filtro vê "domínio normal" e libera. Na nuvem: registre domínio apontando pra `169.254.169.254`.

### Allow list (só libera o que casa) — mais forte, mas quebrável

| Bypass | Exemplo | Por quê |
|---|---|---|
| **Subdomínio** | `https://website.thm.attacker.thm` | Começa com o prefixo, mas o host real é do atacante |
| **Credencial `@`** | `https://website.thm@attacker.com/` | Libs tratam o antes do `@` como user, o depois como host |

> Raiz: validar a **string** em vez de **parsear** a URL. **Nunca use `startsWith()`/regex em URL — use um parser real e cheque o hostname parseado.**

### Open Redirect (encadeamento) — quando os dois falham

```
https://website.thm/link?url=http://169.254.169.254/latest/meta-data/
```
- Allow list satisfeita (começa com `website.thm`). ✅
- Servidor segue o redirect → vai pro metadata. 💀

> **A lição:** encadear dois comportamentos inofensivos sozinhos. Segurança tem que considerar **interações entre features**.

---

## 8. 🎯 Lab prático (Acme avatar + traversal)

**Cenário:** `/private` só acessível pelo próprio servidor (bloqueio por IP). A feature de avatar em `/customers/new-account-page` é o vetor. Combina **campo hidden + bypass de deny list com traversal**.

```text
1) Signup → login → /customers/new-account-page
2) Ctrl+U: cada avatar é um radio button com value = path da imagem
3) Selecionar avatar + Update Avatar → o servidor BUSCA o path,
   lê a resposta e embute em Base64 no src (= Regular SSRF, volta pra você)
4) Inspect no radio → trocar value pra "private" → ERRO (deny list bloqueia /private)
5) Trocar value pra:  x/../private
6) Update Avatar → sucesso
7) Ver source: <img> tem o Base64 do conteúdo de /private
8) Decodificar:  echo "BASE64" | base64 -d   → flag
```

**Por que `x/../private` passa:**

| Estágio | Path | O quê |
|---|---|---|
| Validação (deny list) | `x/../private` | Checa a **string crua** — não começa com `/private` → **passa** ✅ |
| Normalização (servidor) | `/private` | Entra em `x`, sobe com `../`, chega em `/private` 💀 |

> O coração: validação e servidor interpretam o path em **momentos diferentes** (checa antes, normaliza depois). Essa janela é o bug.
> Se `x/../private` não passar, tente URL-encoded: `x/..%2fprivate` ou mais níveis de `../`.

---

## 9. Mitigação

| Defesa | O que faz |
|---|---|
| **Allow list com parsing real** | Compara o **hostname parseado**, não a string |
| **Bloquear IPs privados/loopback/metadata** | Após resolver o DNS, cheque se o IP final é interno (não na string) |
| **Revalidar após redirect** | Bloquear ou revalidar o destino depois de cada redirect |
| **Desabilitar schemes perigosos** | `file://`, `gopher://`, `dict://` fora da allow list |
| **Network segmentation** | O servidor web não deveria alcançar metadata/serviços internos |

> **Deny list é quase sempre fura-fácil** (as formas de IP provam). Prefira allow list **com parser de URL de verdade**.

---

## 10. Erros comuns

- ❌ Confundir CSRF (navegador faz a requisição) com SSRF (servidor faz).
- ❌ Validar URL com `startsWith()`/regex em vez de parser real.
- ❌ Confiar em deny list (bypass com decimal, octal, nip.io, IPv6).
- ❌ Bloquear só na string e não no IP resolvido (nip.io/DNS fura).
- ❌ Esquecer que redirect encadeado fura allow list.
- ❌ No traversal, esquecer que validação ≠ normalização (a janela do bug).
- ❌ Esquecer que DNS callback confirma SSRF mesmo com HTTP bloqueado.
- ❌ Testar sites sem autorização.

---

*Módulo: Web Application Vulnerabilities I — Intro to SSRF. Fecha o quarteto SQLi + CSRF + XSS + SSRF.*
