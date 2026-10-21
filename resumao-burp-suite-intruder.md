# 🛡️ Resumão Burp Suite: Intruder — Anotações TryHackMe

> Terceira sala do módulo Burp Suite. O Intruder **automatiza** o que o Repeater faz à mão: marca posições na requisição e dispara variações de payload a partir de wordlists. Cobre os 4 attack types, brute-force, credential stuffing, IDOR e macros com CSRF token.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, PortSwigger Academy, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [O que é o Intruder](#1-o-que-é-o-intruder)
2. [Positions](#2-positions)
3. [Payloads](#3-payloads)
4. [Os 4 attack types](#4-os-4-attack-types)
5. [Fórmulas de contagem de requisições](#5-fórmulas-de-contagem-de-requisições)
6. [Prática 1: credential stuffing (Pitchfork)](#6-prática-1-credential-stuffing-pitchfork)
7. [Prática 2: IDOR (Sniper + Numbers)](#7-prática-2-idor-sniper--numbers)
8. [Extra-mile: brute-force com CSRF token (Macros)](#8-extra-mile-brute-force-com-csrf-token-macros)
9. [Atalhos e dicas](#9-atalhos-e-dicas)
10. [Erros comuns](#10-erros-comuns)

---

## 1. O que é o Intruder

Ferramenta de **fuzzing/automação**: pega uma requisição capturada e dispara variações a partir de wordlists.

- **Usos:** brute-force de login, fuzzing de diretórios/endpoints/vhosts.
- **Equivalente CLI:** `ffuf`, `Wfuzz`, `hydra`.
- **⚠️ Rate-limited na Community** — bem mais lento que a Pro (pausa entre requisições). Para listas grandes, profissionais usam ffuf/Hydra.
- **Mandar requisição ao Intruder:** `Ctrl + I`.

**4 sub-abas:** Positions · Payloads · Resource Pool (inútil na Community) · Settings.

> **Fuzzing** = testar existência/comportamento aplicando um conjunto de valores a um parâmetro (ex.: `http://alvo/PALAVRA` trocando PALAVRA por cada item da wordlist).

---

## 2. Positions

Define **ONDE** os payloads entram. Marcado em verde entre `§...§`.

| Botão | Função |
|---|---|
| **Add §** | Cria posição a partir do texto selecionado |
| **Clear §** | Remove todas as posições |
| **Auto §** | Readivinha as posições prováveis |

💡 Para brute-force de senha com usuário conhecido: **Clear §** e marque **só** o campo `password`. Sempre limpe os cookies de sessão das posições automáticas.

---

## 3. Payloads

Define **O QUE** entra nas posições. 4 seções:

| Seção | Função |
|---|---|
| **Payload Sets** | Escolhe para qual posição e o tipo (Simple list, Numbers, Dates...) |
| **Payload Settings** | Monta a lista: Add, Paste, Load (arquivo), Clear, Deduplicate |
| **Payload Processing** | Regras por payload: **Add prefix** (início), **Add suffix** (fim), match-and-replace, skip por regex |
| **Payload Encoding** | URL-encode dos caracteres especiais (deixe padrão) |

**Ordem dos payload sets:** cima→baixo, esquerda→direita. Em `username=§...§&password=§...§` → **set 1 = username**, **set 2 = password**.

> Sniper e Battering ram usam **1 lista** (dropdown com 1 opção). Pitchfork e Cluster bomb usam **1 lista por posição**.
> `Add from list` fica acinzentado → é **só da Pro**.

---

## 4. Os 4 attack types

| Tipo | Listas | Como funciona | Quando usar |
|---|---|---|---|
| **Sniper** | 1 | Um payload por vez, **uma posição de cada vez** (as outras ficam no valor original). Nunca combina. | Fuzzing de 1 campo; senha com usuário fixo |
| **Battering ram** | 1 | **Mesmo payload em todas as posições** ao mesmo tempo | Mesmo valor repetido em vários lugares da requisição |
| **Pitchfork** | 1 por posição (máx. 20) | Listas **em paralelo / emparelhadas**: 1º de cada, 2º de cada... | **Credential stuffing** (user+senha já pareados) |
| **Cluster bomb** | 1 por posição (máx. 20) | **Todas as combinações** (produto cartesiano) | Brute-force quando não sabe user nem senha |

**Mnemônico:** Sniper (1 tiro por vez) · Battering ram/aríete (mesma tora em tudo) · Pitchfork/forcado (dentes paralelos) · Cluster bomb (estilhaços em todas as direções).

**⚠️ Pitchfork vs Cluster bomb (o que mais cai):**
- **Pitchfork** emparelha → listas de 3 = **3 requisições** (A1-B1, A2-B2, A3-B3). Para na menor lista.
- **Cluster bomb** combina → listas de 3 = **9 requisições** (tudo contra tudo).

---

## 5. Fórmulas de contagem de requisições

| Attack type | Fórmula | Ex.: 3 palavras, 2 posições |
|---|---|---|
| **Sniper** | palavras × posições | 3 × 2 = **6** |
| **Battering ram** | palavras | **3** |
| **Pitchfork** | tamanho da lista (para na menor) | **3** |
| **Cluster bomb** | lista1 × lista2 × ... | 3 × 3 = **9** |

---

## 6. Prática 1: credential stuffing (Pitchfork)

Alvo: `/support/login`. Falha: sem proteção anti-brute-force. Usamos credenciais vazadas (user já pareado com senha) → **Pitchfork**, não Cluster bomb (100 pares = 100 requisições, em vez de 10.000).

```text
1) wget http://ALVO:9999/Credentials/BastionHostingCreds.zip ; unzip
   (usar usernames.txt e passwords.txt)
2) Capturar POST /support/login no Proxy → Ctrl+I
3) Positions: Clear § → marcar só username e password → Attack type: Pitchfork
4) Payloads: set 1 = usernames.txt | set 2 = passwords.txt
5) Start attack (OK no aviso de rate limit)
6) Todas as respostas são 302 → ordenar por LENGTH
7) A resposta com Length diferente (menor) = login válido
```

> **Lição-chave:** quando o status code não diferencia (tudo 302), use o **Length** da resposta para achar o acerto.

---

## 7. Prática 2: IDOR (Sniper + Numbers)

Alvo: `/support/ticket/NÚMERO` (IDs inteiros e sequenciais).

**IDOR (Insecure Direct Object Reference):** o sistema expõe uma referência direta a um objeto (número do ticket) e **não checa se você tem permissão** de acessá-lo. Trocar o número e ver dado de outro usuário = IDOR.

```text
1) Logado, abrir um ticket e capturar GET /support/ticket/N
   → a requisição PRECISA ter o Cookie de sessão
2) Ctrl+I → Positions: Clear § → marcar só o número → Attack type: Sniper
3) Payloads → Payload type: NUMBERS (From 1, To 50, Step 1)
4) Start attack → ordenar por Length
5) O ticket com status 200 e Length destoante = o alvo (fora do seu usuário)
```

**Detalhe real do lab:** o cookie `session=eyJ...` é Base64. Decodificado dava `{"SupportAuth":"True","SupportID":3}` — ou seja, dava para ver o próprio ID de usuário. O IDOR permitiu ler um ticket que não pertencia ao SupportID 3.

> **Erro que cometi e corrigi:** marcar o `§` na linha errada (no Referer em vez do número) faz todas as respostas virem iguais (mesmo Length). Confira sempre QUAL texto está entre os `§`.

---

## 8. Extra-mile: brute-force com CSRF token (Macros)

Alvo: `/admin/login/`. Agora **com proteção**: um `loginToken` (campo hidden) e um `session` cookie que **mudam a cada requisição**. Reusar valores velhos → **403**. Ataque simples não funciona.

**Solução: Burp Macro** — antes de cada requisição do Intruder, faz um `GET /admin/login/`, extrai o `loginToken` e o `session` novos, e injeta na tentativa de login.

```text
1) Capturar POST /admin/login → Ctrl+I
2) Positions: Clear § → marcar só username e password → Pitchfork
3) Payloads: usernames.txt e passwords.txt

--- Criar o Macro ---
4) Settings (⚙️ topo direito) → categoria Sessions
5) Seção Macros → Add → selecionar um GET /admin/login/ do histórico → OK → nomear → OK

--- Criar a Session Handling Rule ---
6) Sessions → Session Handling Rules → Add
7) Aba Scope:
   - Tools Scope: deixar SÓ Intruder
   - URL Scope: Use suite scope (ou custom scope + http://ALVO/)
8) Aba Details → Rule Actions → Add → "Run a Macro" → escolher o macro
9) Restringir o que o macro atualiza:
   - "Update only the following parameters and headers" → Edit → add loginToken
   - "Update only the following cookies" → Edit → add session
   - OK em tudo

--- Atacar ---
10) Intruder → Start attack
11) Respostas devem ser 302. Se vier 403 → macro não está funcionando (revisar Scope e nomes)
12) Ordenar por Length. Vários tamanhos variam, mas o login certo se destaca por ser bem mais curto
13) Logar com as credenciais (pode precisar dar refresh na página antes)
```

> **Lição-chave:** tokens anti-CSRF que mudam a cada request são a defesa clássica contra brute-force automatizado. Macros + Session Handling Rules são a forma de contornar isso dentro do próprio Burp.

---

## 9. Atalhos e dicas

| Atalho / Ação | Função |
|---|---|
| **Ctrl + I** | Enviar requisição ao Intruder |
| **Ctrl + R** | Enviar ao Repeater |
| Ordenar por **Length** | Achar o acerto quando o status code é igual em tudo |
| **Deduplicate** | Remove payloads repetidos (economiza requisições na Community lenta) |
| **Numbers** payload type | Gera intervalos numéricos sem precisar de wordlist |

---

## 10. Erros comuns

- ❌ Marcar o `§` no lugar errado (ex.: no Referer) → todas as respostas saem iguais.
- ❌ Capturar a requisição **deslogado** no teste de IDOR → sem o cookie de sessão, nada funciona.
- ❌ Usar **Cluster bomb** onde **Pitchfork** bastava → requisições explodem (lista × lista).
- ❌ Esquecer de **Clear §** → o Burp ataca cookies e campos que não interessam.
- ❌ No desafio com CSRF, respostas vindo **403** → macro mal configurado (Scope ou nome do campo errado).
- ❌ Confiar só no **status code** quando ele é igual para todos (302) → use o Length.
- ❌ Carregar wordlists gigantes na Community → Burp trava e o rate limit faz levar horas.
- ❌ Testar sites sem autorização.

---

## Flags / Respostas capturadas

| Task | Resposta |
|---|---|
| Credential stuffing (support) | *(par user:senha da resposta de Length menor)* |
| IDOR (ticket) | Ticket 47 (status 200, Length destoante) |
| Extra-mile (admin + CSRF) | *(par username:password da resposta mais curta)* |

---

*Sala anterior: **Burp Suite: Repeater**. Próxima: **Burp Suite: Other Modules** (Decoder, Comparer, Sequencer).*
