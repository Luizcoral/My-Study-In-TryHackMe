# 🛡️ Resumão Burp Suite: Other Modules — Anotações TryHackMe

> Quarta sala do módulo Burp Suite. Passa por quatro ferramentas de apoio menos conhecidas: **Decoder** (codificar/decodificar/hash), **Comparer** (comparar dados), **Sequencer** (entropia de tokens) e **Organizer** (arquivar requisições). Sala leve, mais exploração de ferramenta que de alvo.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, PortSwigger Academy, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [Decoder](#1-decoder)
2. [Comparer](#2-comparer)
3. [Sequencer](#3-sequencer)
4. [Organizer](#4-organizer)
5. [Atalhos](#5-atalhos)
6. [Como diferenciar as ferramentas](#6-como-diferenciar-as-ferramentas)

---

## 1. Decoder

Canivete suíço de conversão de texto dentro do Burp. Faz 4 coisas:

| Operação | O que é |
|---|---|
| **Decode** | Desembaralha (Base64/URL/hex → texto normal) |
| **Encode** | Embaralha (texto normal → formato que o servidor entende) |
| **Hash** | Gera a "impressão digital" do dado (MD5, SHA...) |
| **Smart Decode** | Tenta adivinhar sozinho como decodificar (= função "Magic" do CyberChef) |

**Interface:** caixa de entrada (1), Text/Hex (2), dropdowns Decode/Encode/Hash (3), Smart Decode (4). O resultado vira uma **caixa nova embaixo**, permitindo **empilhar (stack)** transformações.

### Formatos de encode/decode

| Formato | Exemplo |
|---|---|
| **Plain** | texto cru |
| **URL** | `/` (ASCII 47 → hex 2F) → **`%2F`** |
| **HTML** | `"` → `&#x22;` ou `&quot;` (previne XSS) |
| **Base64** | cookies, tokens |
| **ASCII Hex** | `ASCII` → `4153434949` |
| **Hex/Octal/Binary** | só para números |
| **Gzip** | compressão |

### Hashing (teoria importante)

- **Hashing é mão única (irreversível)** — essa é a diferença para encode/decode.
- Mudança mínima no input muda o hash todo: `MD5sum` → `4ae1a02de5bd02a5515f583f4fca5e8c`, `MD5SUM` → `13b436b09172400c9eb2f69fbd20adad`.
- **Usos:** integridade de arquivos e armazenamento seguro de senhas (guarda o hash, não a senha).
- ⚠️ **MD5 está obsoleto** — use bcrypt/argon2/SHA-256+.

**Gerar hash no Decoder:** digite o texto → dropdown **Hash → MD5** (sai em Hex view, binário) → **Encode as → ASCII Hex** (vira a string hex legível).

> 💡 URL encoding é o mesmo `%2F`/`%27`/`%20` do `Ctrl+U` no Repeater. Um cookie Base64 pode ir direto por *Send to Decoder → Decode as Base64*.

---

## 2. Comparer

Compara dois dados e destaca as diferenças, por **words** ou **bytes**.

**Tela de carga:** tabelas à esquerda (selecione 2 datasets), botões Paste/Load/Remove/Clear, e os botões **Words / Bytes**. Também via botão direito → **Send to Comparer**.

**Tela de resultado (pop-up):**
- Dados lado a lado (texto ou hex).
- **Comparison key:** legenda de cores — **modified**, **deleted**, **added**.
- **Sync views:** sincroniza o formato dos dois lados.
- O **título** mostra o total de diferenças.

> 💡 **Uso real:** comparar a resposta de "login falhou" com a de "login certo" para ver exatamente o que muda — o passo seguinte a ordenar por Length no Intruder. Serve também para enumeração ("usuário existe / não existe").

---

## 3. Sequencer

Mede a **entropia (aleatoriedade)** de **tokens** (cookies de sessão, CSRF tokens, tokens de reset de senha).

**Por que importa:** token previsível = atacante adivinha os próximos valores. Token de reset previsível = sequestro de conta.

**Dois modos:**

| Modo | Como funciona |
|---|---|
| **Live Capture** (padrão) | Repete a requisição milhares de vezes, coletando tokens para análise |
| **Manual Load** | Carrega uma lista de tokens já coletados (evita barulho de milhares de requisições) |

### Lendo o relatório (Summary)

| Campo | Significado |
|---|---|
| **Overall result** | Veredito geral de segurança |
| **Effective entropy** | Aleatoriedade em bits — **117 bits** = alto = seguro |
| **Reliability** | Nível de significância 1% = **99% de confiança** |
| **Sample** | Detalhes da amostra analisada |

> **Regra:** mais bits de entropia = mais aleatório = mais seguro. Entropia baixa = token previsível = inseguro.
> O relatório é **forte indicador**, não prova definitiva (estatística tem incerteza).

> 💡 **Conexão:** é a ferramenta que avalia se aquele `loginToken` CSRF do extra-mile do Intruder é realmente seguro. Entropia baixa → dava para prever e dispensar o macro.

---

## 4. Organizer

Guarda e anota **cópias read-only** de requisições HTTP para revisitar depois. É o "caderno de favoritos" do pentest.

- **Usos:** marcar requisições para investigar, já identificadas como vulneráveis, ou para o relatório final.
- **Enviar:** botão direito → **Send to Organizer** ou **Ctrl+O**.
- Cada item é uma **cópia somente leitura**, congelada no momento do envio (não edita nem reenvia).
- Tabela com índice, horário, status, módulo de origem, método, host, path, status code, tamanho e notas.

---

## 5. Atalhos

| Atalho | Módulo |
|---|---|
| **Ctrl + R** | Repeater |
| **Ctrl + I** | Intruder |
| **Ctrl + O** | Organizer |
| Botão direito → Send to... | Decoder / Comparer / Sequencer |

---

## 6. Como diferenciar as ferramentas

| Ferramenta | Função em uma linha |
|---|---|
| **Decoder** | Converte formatos (encode/decode/hash) |
| **Comparer** | Mostra a diferença entre dois dados |
| **Sequencer** | Avalia se um token é aleatório o suficiente |
| **Organizer** | Arquiva requisições (read-only) para depois |
| *(Repeater)* | Testa e reenvia (editável) — ≠ Organizer |

---

*Sala anterior: **Burp Suite: Intruder**. Próxima (última do módulo): **Burp Suite: Extensions**.*
