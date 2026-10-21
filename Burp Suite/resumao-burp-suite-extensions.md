# 🛡️ Resumão Burp Suite: Extensions — Anotações TryHackMe

> Quinta e última sala do módulo Burp Suite. Teórica e curta: como estender o Burp com módulos adicionais via BApp Store, integrar Python com Jython e visão geral da API. Não precisa de lab machine.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, PortSwigger Academy, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [Interface de Extensions](#1-interface-de-extensions)
2. [BApp Store](#2-bapp-store)
3. [Jython (extensões em Python)](#3-jython-extensões-em-python)
4. [A API do Burp](#4-a-api-do-burp)
5. [Pontos de prova](#5-pontos-de-prova)

---

## 1. Interface de Extensions

| Componente | Função |
|---|---|
| **Extensions List** | Lista das extensões instaladas; ativa/desativa cada uma |
| **Add** | Instala extensão de um arquivo em disco (custom ou externa) |
| **Remove** | Desinstala a extensão selecionada |
| **Up / Down** | Ordem de execução das extensões |
| **Details / Output / Errors** | Info, saída e erros da extensão (debug) |

> ⚠️ **A ordem (Up/Down) importa:** extensões rodam **de cima para baixo**. Quando duas modificam a mesma requisição, a ordem evita que uma interfira na outra.

---

## 2. BApp Store

Loja **oficial** de extensões do Burp (Burp App Store).

- Maioria escrita em **Java** (integra automaticamente) ou **Python** (precisa do **Jython**).
- Instalar: sub-aba **BApp Store** → buscar → **Install**.
- Extensões podem adicionar uma **nova aba** no topo do Burp ou itens no **menu de botão direito**.

**Exemplo da sala — Request Timer** (de Nick Taylor):
- Registra o **tempo de resposta** de cada requisição.
- Serve para **vulnerabilidades time-based**: se um login demora mais com usuário válido do que com inválido, dá para enumerar usuários pela diferença de tempo.

---

## 3. Jython (extensões em Python)

- Para rodar extensões em **Python**, o Burp precisa do **Jython** = implementação Java do Python.
- Baixar o **Jython Standalone JAR** do site oficial.
- Configurar em: **Extensions → Settings → Python environment → Location of Jython standalone JAR file**.
- Mesmo processo em qualquer SO (Java é multiplataforma).
- 💡 **No AttackBox já vem configurado** — pode pular essa etapa.

---

## 4. A API do Burp

- Expõe as funcionalidades do Burp para quem quer **escrever extensões próprias**.
- Visível na sub-aba **APIs** do módulo Extensions (cada item = um endpoint da API).
- **Três linguagens suportadas:**

| Linguagem | Como |
|---|---|
| **Java** | Nativo |
| **Python** | Via **Jython** |
| **Ruby** | Via **JRuby** |

- Codar extensão foge do escopo da sala. Referência oficial: documentação da PortSwigger ("Writing your first Burp Suite extension").

---

## 5. Pontos de prova

- Extensões rodam **de cima para baixo** (ordem Up/Down importa).
- **BApp Store** = loja oficial.
- **Java** integra sozinho; **Python** precisa de **Jython**; **Ruby** precisa de **JRuby**.
- **Request Timer** = extensão de exemplo, para vulnerabilidades **time-based**.
- **Add** instala de disco; **Remove** desinstala.

---

## ✅ Módulo Burp Suite completo

| Sala | Foco |
|---|---|
| The Basics | Framework, edições, Proxy |
| Repeater | Editar/reenviar requisições; SQLi manual |
| Intruder | Ataques automatizados; brute-force, IDOR, macros/CSRF |
| Other Modules | Decoder, Comparer, Sequencer, Organizer |
| **Extensions** | BApp Store, Jython, API |

---

*Sala anterior: **Burp Suite: Other Modules**. Módulo Burp Suite finalizado 🎉*
