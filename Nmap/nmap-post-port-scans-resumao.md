# 🛰️ Resumão Nmap Post Port Scans — Anotações TryHackMe

> Última sala da série Nmap. Cobre o que vem *depois* de achar as portas: versões de serviço, fingerprint de SO, traceroute, NSE e como salvar os resultados.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [Visão geral do fluxo](#1-visão-geral-do-fluxo)
2. [Service Detection (`-sV`)](#2-service-detection--sv)
3. [OS Detection (`-O`)](#3-os-detection--o)
4. [Traceroute (`--traceroute`)](#4-traceroute---traceroute)
5. [Nmap Scripting Engine (NSE)](#5-nmap-scripting-engine-nse)
6. [Salvando a saída](#6-salvando-a-saída)
7. [O atalho `-A`](#7-o-atalho--a)
8. [Tabela-resumo da sala](#8-tabela-resumo-da-sala)
9. [Fluxo de trabalho completo (as 4 salas)](#9-fluxo-de-trabalho-completo-as-4-salas)
10. [Erros comuns](#10-erros-comuns)

---

## 1. Visão geral do fluxo

A série completa do Nmap no TryHackMe segue 9 etapas:

```text
1. Enumerate targets     → Live Host Discovery
2. Discover live hosts   → Live Host Discovery
3. Reverse-DNS lookup    → Live Host Discovery
4. Scan ports            → Basic / Advanced Port Scans
5. Detect versions       → esta sala (-sV)
6. Detect OS             → esta sala (-O)
7. Traceroute            → esta sala (--traceroute)
8. Scripts                → esta sala (NSE)
9. Write output           → esta sala (-oN/-oG/-oX/-oA)
```

Esta sala cobre as etapas **5 a 9**: depois de já saber *quais* portas estão abertas, o foco vira *o que* roda nelas, *qual sistema* é o alvo, *o caminho* até ele, e *como registrar* tudo.

---

## 2. Service Detection (`-sV`)

```bash
sudo nmap -sV 10.64.173.123
```

Acrescenta a coluna **VERSION**, com a versão exata do serviço (ex.: `OpenSSH 9.2p1 Debian 2+deb12u3`).

### Controlando a intensidade

```bash
nmap -sV --version-intensity LEVEL   # 0 (mais leve) a 9 (mais completo)
nmap -sV --version-light             # atalho = intensidade 2
nmap -sV --version-all               # atalho = intensidade 9
```

### O ponto mais importante

> **`-sV` exige completar o 3-way handshake.** Para pegar a versão, o Nmap precisa se conectar de verdade e coletar o banner do serviço — por isso **não existe stealth (`-sS`) combinado com `-sV`** nas portas que serão sondadas; o Nmap completa a conexão mesmo pedindo `-sS` junto.

### SERVICE (chute) vs. VERSION (confirmado)

| Coluna | Como é descoberta |
|---|---|
| **SERVICE** | Chute baseado no número da porta, sem precisar conectar — pode errar (ex.: porta 5000 marcada como `upnp` sendo na verdade uma app Flask) |
| **VERSION** | Confirmado via conexão real e coleta do banner |

Uma linha pode aparecer sem versão (ex.: `rpcbind` sem detalhes) quando o Nmap identifica o serviço mas não extrai uma versão específica do banner.

**Por que importa:** a versão exata é o ponto de partida para buscar CVEs e vulnerabilidades conhecidas daquele serviço.

---

## 3. OS Detection (`-O`)

```bash
sudo nmap -sS -O 10.64.173.123
```

Analisa **como o alvo se comporta na pilha TCP/IP** e compara com um banco de assinaturas.

### O que o Nmap olha

| Sinal | O que revela |
|---|---|
| **TTL** | Linux costuma responder 64, Windows 128 |
| **TCP Sequence** | Como o número de sequência é gerado/incrementado, varia por SO/kernel |
| **Banners e particularidades dos serviços** | Diferenças sutis entre famílias de SO |

### "No exact OS matches for host"

É um resultado **comum e esperado**, não um erro — o fingerprint coletado não bateu perfeitamente com nenhuma assinatura do banco. Mesmo assim, o Nmap despeja o fingerprint cru (`TCP/IP fingerprint:`) e, às vezes, uma lista de palpites:

```text
Running (JUST GUESSING): Linux 5.X|6.X|4.X (96%), Google Android 10.X|11.X|12.X (93%)
```

- **"JUST GUESSING"** avisa que é aproximação.
- **Android aparecer é normal**: roda sobre kernel Linux, então a pilha TCP/IP se parece com a de um Linux comum — não é erro do Nmap.
- A porcentagem é a confiança do palpite, não uma garantia.
- O resultado pode **variar entre execuções** (jitter de rede, timing), inclusive mostrando ou não a lista de palpites.

### Por que não bate exatamente

Virtualização, firewall filtrando/alterando pacotes, camadas de rede de nuvem, kernel customizado, middleboxes no caminho.

> **Lição:** `-O` é probabilístico, não garantido. Combine com outras fontes — a versão do SSH pego via `-sV` (ex.: `Debian 2+deb12u3`) costuma ser muito mais confiável e específica que o `-O` sozinho.

---

## 4. Traceroute (`--traceroute`)

```bash
nmap -sS --traceroute 10.64.173.123
```

Mostra os **roteadores no caminho** até o alvo.

### Diferença para o traceroute tradicional

| | Traceroute padrão (Linux/macOS/Windows) | Nmap `--traceroute` |
|---|---|---|
| TTL inicial | Baixo, vai subindo | Alto, vai descendo |

### Lendo o resultado

```text
HOP RTT       ADDRESS
1   124.45 ms 192.168.128.1
2   ...
3   125.16 ms 10.64.173.123
```

- Hops sem resposta (`...`) são normais: **muitos roteadores não enviam** a mensagem ICMP "Time-to-Live exceeded", então o Nmap não descobre o IP daquele salto.
- A porta usada na sondagem pode variar entre execuções — o Nmap escolhe automaticamente uma porta provavelmente não-obstruída.

---

## 5. Nmap Scripting Engine (NSE)

### O que é

Interpretador de **Lua** embutido no Nmap. Scripts (`.nse`) estendem o que o Nmap faz, sem precisar programar — só escolher e rodar. Uma instalação padrão traz **~600 scripts**, organizados por protocolo no nome:

```bash
ls /usr/share/nmap/scripts
ls /usr/share/nmap/scripts | grep http    # ex.: mais de 130 scripts só de HTTP
```

### Categorias

```bash
nmap -sC <alvo>               # roda a categoria "default"
nmap --script=default <alvo>  # mesma coisa, forma explícita
```

| Categoria | O que faz |
|---|---|
| `default` (`-sC`) | Conjunto seguro e informativo |
| `safe` | Não deve crashar o alvo |
| `discovery` | Extrai informações acessíveis (tabelas de banco, nomes DNS...) |
| `version` | Ajuda a identificar versões |
| `vuln` | Verifica vulnerabilidades conhecidas |
| `auth` | Scripts relacionados a autenticação |
| `brute` | Força bruta contra logins |
| `exploit` | Tenta explorar vulnerabilidades |
| `dos` | Testa suscetibilidade a negação de serviço |
| `intrusive` | Agressivos (brute-force, exploração) — podem derrubar o serviço |
| `malware` | Procura backdoors |
| `broadcast` | Descobre hosts via mensagens de broadcast |
| `external` | Consulta serviços de terceiros (ex.: VirusTotal) |
| `fuzzer` | Ataques de fuzzing |

⚠️ Um script pode pertencer a mais de uma categoria. `brute`, `dos`, `exploit` e `intrusive` **podem derrubar ou comprometer** o serviço — escolha com cuidado.

### O que o `-sC` revela na prática

```bash
sudo nmap -sS -sC 10.64.173.123
```

| Porta/serviço | O que o NSE extraiu |
|---|---|
| SSH | Chaves públicas do servidor (ECDSA, ED25519) |
| SMTP/POP3/IMAP | Certificado TLS (validade), capacidades do protocolo |
| HTTP | Título da página (`http-title`) — pode indicar página default/nunca configurada |
| rpcbind | Todos os serviços RPC registrados, com portas e versões (`rpcinfo`) |

> O NSE **complementa** o que já foi descoberto com `-sV`/`-O` — não troca a versão, acrescenta contexto em cima dela.

### Rodando um script específico

```bash
nmap --script "SCRIPT-NAME" <alvo>
nmap --script "ftp*" <alvo>              # wildcard, pega todos que começam com "ftp"
sudo nmap -sS -n --script "http-date" 10.64.173.123
```

O `-n` desliga a resolução de DNS reverso (só acelera, sem relação com o NSE).

### Antes de rodar um script desconhecido

```bash
less /usr/share/nmap/scripts/ftp-brute.nse
```

O cabeçalho do arquivo traz uma descrição em texto simples do que o script faz.

### Scripts de terceiros

> **Não rode script de autor em quem você não confia.** É código executado com seus privilégios contra o alvo.

### Exemplo prático: `ssh2-enum-algos`

```bash
sudo nmap -sS -n --script "ssh2-enum-algos" 10.64.173.123
```

Lista os algoritmos oferecidos pelo servidor SSH2, separados por categoria:

```text
kex_algorithms: (negociação de chave)
server_host_key_algorithms: (identidade do servidor)
encryption_algorithms:
mac_algorithms:
compression_algorithms:
```

Útil para auditar se o servidor ainda aceita algoritmos fracos/obsoletos.

---

## 6. Salvando a saída

Depois do scan, salve o resultado em vez de deixar só na tela. Existem 4 formatos.

### Normal (`-oN`)

```bash
nmap -oN scan.nmap 10.64.173.123
```

Exatamente o que aparece na tela, salvo em arquivo. Fácil de ler; ruim para filtrar entre vários hosts.

### Grepable (`-oG`)

```bash
nmap -oG scan.gnmap 10.64.173.123
```

Pensado para `grep`. Cada host vira **uma linha só**, longa, mas **completa** (IP + portas + serviços juntos).

**Por que importa:**

```bash
grep http scan.nmap     # formato normal
```
```text
80/tcp  open  http    nginx 1.6.2
```
Essa linha **não diz de qual host veio** — inútil ao escanear várias máquinas de uma vez.

```bash
grep http scan.gnmap    # formato grepable
```
```text
Host: 10.64.173.123  Ports: 22/open/tcp//ssh//...  80/open/tcp//http//nginx 1.6.2/...
```
Aqui o IP vem **junto na mesma linha** — o resultado do grep já é autossuficiente.

### XML (`-oX`)

```bash
nmap -oX scan.xml 10.64.173.123
```

Formato estruturado, para **outros programas lerem** (não para leitura humana direta): conversão para HTML (`xsltproc`), import no Metasploit, parsing com Python, ferramentas como Faraday/Dradis.

### Todos juntos (`-oA`)

```bash
nmap -oA scan_debra2 10.64.173.123
```

Gera os três de uma vez: `scan_debra2.nmap`, `.gnmap`, `.xml`. **Prática recomendada**, já que custa pouco e cobre qualquer necessidade futura.

Dica de nomenclatura (evita o problema de arquivos se acumulando e ficando difícil achar o antigo):

```bash
sudo nmap -sS -sV -sC -p- 10.64.173.123 -oA $(date +%Y%m%d)_alvo_fullscan
```

### Script Kiddie (`-oS`) — não recomendado

```bash
nmap -sS 127.0.0.1 -oS saida.kiddie
```

Pega o texto normal e deforma a grafia (`$tart!ng nMaP`, `h0St !s uP`). É uma piada/easter egg do Nmap, **sem utilidade prática** — inútil para `grep`, histórico ou qualquer uso sério.

---

## 7. O atalho `-A`

```bash
sudo nmap -A 10.64.173.123
```

Combina tudo que esta sala ensinou num único comando:

```text
-A  =  -sV + -O + -sC + --traceroute
```

De uma vez: versão dos serviços, detecção de SO, scripts default do NSE e traceroute. É o "modo completo" de reconhecimento.

---

## 8. Tabela-resumo da sala

| Opção | Significado |
|---|---|
| `-sV` | Determina serviço/versão nas portas abertas |
| `-sV --version-light` | Sondas mais prováveis (intensidade 2) |
| `-sV --version-all` | Todas as sondas disponíveis (intensidade 9) |
| `-O` | Detecta o SO |
| `--traceroute` | Mapeia o caminho até o alvo |
| `--script=SCRIPTS` | Escolhe scripts NSE específicos |
| `-sC` / `--script=default` | Roda os scripts da categoria default |
| `-A` | Atalho para `-sV -O -sC --traceroute` |
| `-oN` | Salva em formato normal |
| `-oG` | Salva em formato grepable |
| `-oX` | Salva em formato XML |
| `-oA` | Salva nos três formatos de uma vez |

---

## 9. Fluxo de trabalho completo (as 4 salas)

```text
1. Live Host Discovery
   nmap -sn <rede>/24                       → quem está vivo

2. Basic Port Scans
   sudo nmap -sS -p- --min-rate 1000 <alvo> → quais portas estão abertas

3. Advanced Port Scans (quando há firewall/IDS no caminho)
   sudo nmap -sA <alvo>                     → mapear regras de firewall
   sudo nmap -sI ZOMBIE_IP <alvo>           → scan furtivo via zumbi

4. Post Port Scans (esta sala)
   sudo nmap -sV -sC <alvo>                 → o que roda e detalhes
   sudo nmap -O --traceroute <alvo>         → SO e caminho de rede
   sudo nmap -A -oA relatorio_final <alvo>  → tudo junto, salvo em 3 formatos
```

---

## 10. Erros comuns

- ❌ Usar `-sS` esperando furtividade junto com `-sV` — `-sV` sempre completa a conexão.
- ❌ Tratar `-O` como resposta definitiva — é probabilístico, confirme com outras fontes (ex.: banner do `-sV`).
- ❌ Estranhar "No exact OS matches" ou Android aparecer nos palpites — é comportamento normal do fingerprinting.
- ❌ Rodar scripts de categorias agressivas (`brute`, `exploit`, `dos`, `intrusive`) sem entender o que fazem.
- ❌ Baixar e rodar scripts NSE de autores não confiáveis.
- ❌ Salvar só em formato normal quando o scan cobre várias máquinas — prefira `-oG` ou `-oA` nesses casos.
- ❌ Usar `-oS` (Script Kiddie) achando que serve para algo sério.
- ❌ Esquecer `-oA` em scans demorados ou importantes — depois é tarde pra reconstruir o resultado.

---

## 📚 Referências

- [Documentação oficial do Nmap](https://nmap.org/book/man.html)
- [Nmap Reference Guide](https://nmap.org/book/toc.html)
- [TryHackMe – Nmap Post Port Scans](https://tryhackme.com/room/nmap04)
- [NSE Script Library](https://nmap.org/nsedoc/)
