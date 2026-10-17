[nmap-resumao.md](https://github.com/user-attachments/files/33441845/nmap-resumao.md)
# 🛰️ Resumão Nmap & Scanning de Portas

> Anotações de estudo (TryHackMe + complementos) para o repositório de Cybersecurity.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.** Escanear redes sem permissão pode ser crime.

---

## 📑 Índice

1. [O que é o Nmap](#1-o-que-é-o-nmap)
2. [Estados das portas](#2-estados-das-portas)
3. [Flags TCP e 3-way handshake](#3-flags-tcp-e-3-way-handshake)
4. [Tipos de varredura](#4-tipos-de-varredura)
5. [Descoberta de hosts](#5-descoberta-de-hosts)
6. [Escolhendo as portas](#6-escolhendo-as-portas)
7. [Velocidade e temporização](#7-velocidade-e-temporização)
8. [Serviços, versões e SO](#8-serviços-versões-e-so)
9. [NSE – Nmap Scripting Engine](#9-nse--nmap-scripting-engine)
10. [Salvando resultados](#10-salvando-resultados)
11. [Firewall e evasão (conceitos)](#11-firewall-e-evasão-conceitos)
12. [Fluxo de trabalho recomendado](#12-fluxo-de-trabalho-recomendado)
13. [Cheat sheet final](#13-cheat-sheet-final)
14. [Erros comuns](#14-erros-comuns)

---

## 1. O que é o Nmap

**Nmap (Network Mapper)** é uma ferramenta para:

- descobrir **hosts ativos** em uma rede;
- identificar **portas abertas**;
- detectar **serviços e versões**;
- estimar o **sistema operacional**;
- rodar **scripts** de enumeração e verificação (NSE).

```bash
nmap 10.10.10.10
```

📌 Por padrão, o Nmap verifica as **1.000 portas TCP mais comuns** (não todas as 65.535).

---

## 2. Estados das portas

| Estado | Significado |
|---|---|
| `open` | Há um serviço aceitando conexões naquela porta. |
| `closed` | A porta responde, mas nenhum serviço está escutando. |
| `filtered` | Firewall/filtro impede saber se está aberta ou fechada (sem resposta). |
| `unfiltered` | Porta alcançável, mas não dá para saber se está aberta ou fechada (típico do `-sA`). |
| `open\|filtered` | Não foi possível distinguir entre aberta e filtrada. |
| `closed\|filtered` | Não foi possível distinguir entre fechada e filtrada. |

💡 **Importante**
- Porta `closed` **não** significa host desligado: o host respondeu.
- Falta de resposta **não** prova que o host está offline.
- `filtered` costuma indicar **firewall** no caminho.

---

## 3. Flags TCP e 3-way handshake

### Flags

| Flag | Função |
|---|---|
| `SYN` | Inicia uma conexão. |
| `ACK` | Confirma recebimento de dados ou de etapa da conexão. |
| `RST` | Reinicia/interrompe a conexão. |
| `FIN` | Indica que o emissor terminou de enviar dados. |
| `PSH` | Pede entrega imediata dos dados à aplicação. |
| `URG` | Indica que o ponteiro de urgência é relevante. |

### 3-way handshake

```text
Cliente                Servidor
   |  ---- SYN ---->      |
   |  <-- SYN/ACK ---     |
   |  ---- ACK ---->      |
   |   Conexão estabelecida
```

### Como o servidor responde a um SYN

| Situação da porta | Resposta |
|---|---|
| Aberta | `SYN/ACK` |
| Fechada | `RST` (ou `RST/ACK`) |
| Filtrada | Nenhuma resposta ou erro ICMP |

---

## 4. Tipos de varredura

| Opção | Nome | Para que serve |
|---|---|---|
| `-sT` | TCP Connect | Completa o handshake. Não exige root. |
| `-sS` | SYN Scan ("half-open") | Envia SYN e interrompe com RST. Rápido; exige root. **Padrão com privilégios.** |
| `-sU` | UDP Scan | Verifica portas UDP (lento). |
| `-sA` | ACK Scan | Mapeia regras de firewall; **não** diz se a porta está aberta. |
| `-sF` | FIN Scan | Envia pacotes com FIN. |
| `-sN` | NULL Scan | Envia pacotes sem nenhuma flag. |
| `-sX` | Xmas Scan | Envia FIN + PSH + URG. |

### `-sT` vs `-sS`

```text
-sT:  SYN → SYN/ACK → ACK → (encerra conexão)
-sS:  SYN → SYN/ACK → RST
```

| | `-sT` | `-sS` |
|---|---|---|
| Handshake completo | ✅ | ❌ |
| Precisa de root/sudo | ❌ | ✅ (pacotes brutos) |
| Aparece em logs de aplicação | Mais provável | Menos provável (mas ainda detectável por IDS/firewall) |
| Quando usar | Sem privilégios | Padrão em pentest/CTF |

> Sem root, o Nmap usa `-sT` automaticamente.

### Como interpretar FIN / NULL / Xmas

Esses scans exploram o comportamento da RFC do TCP:

| Resposta | Interpretação |
|---|---|
| `RST` | Porta **closed** |
| Sem resposta | Porta **open\|filtered** |
| ICMP unreachable | Porta **filtered** |

⚠️ Em **Windows** e alguns dispositivos, o comportamento foge da RFC (todas as portas respondem RST), então o resultado não é confiável.

### Como interpretar o ACK Scan

| Resposta | Estado |
|---|---|
| `RST` | `unfiltered` (o pacote passou pelo firewall) |
| Sem resposta / ICMP | `filtered` |

### Como interpretar o UDP Scan

UDP não tem handshake, então:

| Resposta | Estado |
|---|---|
| Resposta UDP | `open` |
| ICMP *port unreachable* | `closed` |
| Sem resposta | `open\|filtered` |

📌 UDP é **lento** (muitos timeouts). Dica: combine com `--top-ports`.

```bash
sudo nmap -sU --top-ports 20 10.10.10.10
```

---

## 5. Descoberta de hosts

Descobrir **quem está vivo** sem varrer todas as portas.

| Opção | Função |
|---|---|
| `-sn` | Descoberta de hosts **sem** port scan (antigo "ping scan"). |
| `-PR` | Sondagem **ARP** (rede local Ethernet). |
| `-PS` | Sondagem TCP **SYN**. |
| `-PA` | Sondagem TCP **ACK**. |
| `-PU` | Sondagem **UDP**. |
| `-PE` | ICMP Echo (ping tradicional). |
| `-Pn` | **Pula** a descoberta: trata todos os hosts como ativos. |
| `-n` | Não faz resolução DNS (mais rápido). |
| `--reason` | Mostra por que o Nmap classificou host/porta daquele jeito. |

```bash
nmap -sn 10.10.10.0/24
nmap -sn --reason 10.10.10.0/24
nmap -PS -sn 10.10.10.0/24
sudo nmap -PU -sn 10.10.10.0/24
nmap -sn -n 192.168.0.0/24
```

### Comportamento padrão do `-sn`

- **Rede local** (mesma sub-rede, com root): usa **ARP**.
- **Rede remota**: usa ICMP Echo, TCP SYN na 443, TCP ACK na 80 e ICMP Timestamp.

### Ping vs. SYN Scan vs. `-PS`

| Ferramenta | O que faz |
|---|---|
| `ping` | Usa ICMP para ver se o host responde. |
| `-sS` | Verifica portas TCP específicas. |
| `-PS` | Usa SYN só para **descobrir hosts vivos**. |

> Um host pode bloquear ping e ainda ter portas abertas. Se o host parece "morto" mas você suspeita que não está, use `-Pn`.

---

## 6. Escolhendo as portas

| Opção | O que faz |
|---|---|
| `-p22,80,443` | Somente as portas listadas. |
| `-p20-25` | Intervalo de 20 a 25 (inclusive). |
| `-p1-1023` | Portas "well-known". |
| `-p-` | **Todas** as 65.535 portas TCP. |
| `-F` | As 100 portas mais comuns (modo *fast*). |
| `--top-ports 10` | As 10 portas mais comuns. |
| `-r` | Ordem numérica (padrão é ordem aleatória). |
| `-p U:53,T:80` | Mistura portas UDP e TCP. |
| `--open` | Mostra **apenas** portas abertas. |

```bash
nmap -sS -p22,80,443 10.10.10.10
sudo nmap -sS -p- 10.10.10.10
nmap -sT -F 10.10.10.10
nmap --top-ports 10 10.10.10.10
```

### Portas comuns para memorizar

| Porta | Serviço | | Porta | Serviço |
|---|---|---|---|---|
| 21 | FTP | | 445 | SMB |
| 22 | SSH | | 1433 | MSSQL |
| 23 | Telnet | | 3050 | Firebird |
| 25 | SMTP | | 3306 | MySQL |
| 53 | DNS (TCP/UDP) | | 3389 | RDP |
| 80 | HTTP | | 5432 | PostgreSQL |
| 110 | POP3 | | 5900 | VNC |
| 111 | RPCbind | | 6379 | Redis |
| 135 | MS-RPC | | 8080 | HTTP alternativo |
| 139 | NetBIOS | | 8443 | HTTPS alternativo |
| 143 | IMAP | | 27017 | MongoDB |
| 161 | SNMP (UDP) | | 1521 | Oracle |
| 443 | HTTPS | | | |

> Essas associações são **convenções**. Um serviço pode rodar em qualquer porta, por isso `-sV` é importante.

---

## 7. Velocidade e temporização

### Perfis `-T`

| Opção | Perfil | Característica |
|---|---|---|
| `-T0` | Paranoid | Extremamente lento (evasão de IDS). |
| `-T1` | Sneaky | Muito lento. |
| `-T2` | Polite | Gasta menos banda/CPU do alvo. |
| `-T3` | Normal | **Padrão**. |
| `-T4` | Aggressive | Rápido; comum em labs/CTFs com boa conexão. |
| `-T5` | Insane | Muito agressivo; pode perder portas e causar instabilidade. |

```bash
nmap -sS -T4 10.10.10.10
```

> Um perfil lento **não** garante invisibilidade. Firewalls e IDS ainda podem detectar.

### Taxa e paralelização

| Opção | Função |
|---|---|
| `--min-rate 100` | Pede mínimo de ~100 pacotes/s. |
| `--max-rate 10` | Limita a ~10 pacotes/s. |
| `--min-parallelism 10` | Mínimo de 10 sondagens simultâneas. |
| `--max-parallelism 20` | Máximo de 20 sondagens simultâneas. |
| `--max-retries 1` | Reduz retransmissões (mais rápido, menos preciso). |
| `--host-timeout 5m` | Desiste de um host após 5 minutos. |

- **Rate** = pacotes por segundo.
- **Parallelism** = sondagens simultâneas.

---

## 8. Serviços, versões e SO

| Opção | Função |
|---|---|
| `-sV` | Detecta serviço e **versão** nas portas abertas. |
| `--version-intensity 0-9` | Define o esforço da detecção de versão. |
| `-O` | Tenta detectar o **sistema operacional** (exige root). |
| `-sC` | Roda os scripts NSE **default**. |
| `-A` | Agressivo: `-sV` + `-O` + `-sC` + `traceroute`. |

```bash
nmap -sV 10.10.10.10
sudo nmap -O 10.10.10.10
sudo nmap -sS -sV -sC -p- 10.10.10.10
sudo nmap -A 10.10.10.10
```

💡 Versão encontrada → pesquise CVEs e exploits (ex.: `searchsploit "OpenSSH 7.2"`).

---

## 9. NSE – Nmap Scripting Engine

Scripts em Lua para enumeração, detecção de vulnerabilidades e mais.

| Opção | Função |
|---|---|
| `-sC` | Scripts da categoria `default`. |
| `--script=<nome>` | Roda um script específico. |
| `--script=<categoria>` | Roda uma categoria (`vuln`, `safe`, `discovery`, `auth`...). |
| `--script-help=<nome>` | Mostra a ajuda do script. |

```bash
nmap --script=http-title -p80 10.10.10.10
nmap --script=vuln 10.10.10.10
nmap --script "smb-enum-*" -p445 10.10.10.10
nmap --script=ftp-anon -p21 10.10.10.10
```

📂 Scripts ficam em `/usr/share/nmap/scripts/` (Linux).

```bash
ls /usr/share/nmap/scripts | grep http
```

⚠️ Categorias como `intrusive`, `brute` e `dos` podem **derrubar serviços** ou gerar muito ruído. Cuidado.

---

## 10. Salvando resultados

| Opção | Formato |
|---|---|
| `-oN arquivo.txt` | Saída normal (legível). |
| `-oX arquivo.xml` | XML (para ferramentas). |
| `-oG arquivo.gnmap` | Grepable (legado). |
| `-oA nome` | **Os três** de uma vez. |
| `-v` / `-vv` | Mais verbosidade (mostra portas abertas em tempo real). |
| `-d` | Modo debug. |

```bash
sudo nmap -sS -sV -p- -oA scan_alvo 10.10.10.10
```

> Salve **sempre** os resultados. Rodar `-p-` de novo só porque esqueceu de salvar é perda de tempo.

---

## 11. Firewall e evasão (conceitos)

Para estudo e laboratório. Entender isso ajuda tanto no ataque quanto na defesa.

| Opção | Ideia |
|---|---|
| `-f` | Fragmenta os pacotes IP. |
| `--mtu 16` | Define o tamanho dos fragmentos (múltiplo de 8). |
| `-D RND:5` | Usa *decoys* (iscas) para misturar o IP real. |
| `-S <ip>` | Falsifica o IP de origem (você não recebe a resposta). |
| `--source-port 53` | Usa uma porta de origem específica. |
| `--data-length 25` | Adiciona dados aleatórios aos pacotes. |
| `-Pn` | Evita a descoberta quando ICMP é bloqueado. |
| `-sA` | Ajuda a mapear regras de firewall. |

📌 Nenhuma técnica torna você "invisível". IDS/IPS modernos detectam muitas dessas tentativas.

---

## 12. Fluxo de trabalho recomendado

```text
1) Descobrir hosts vivos
   nmap -sn 10.10.10.0/24

2) Scan rápido de todas as portas
   sudo nmap -sS -p- --min-rate 1000 -T4 -oN todas.txt 10.10.10.10

3) Scan detalhado só nas portas abertas
   sudo nmap -sS -sV -sC -p22,80,445 -oA detalhado 10.10.10.10

4) Scripts específicos conforme o serviço
   nmap --script=vuln -p80 10.10.10.10

5) UDP nas principais portas
   sudo nmap -sU --top-ports 20 10.10.10.10

6) Documentar e pesquisar versões/CVEs
```

💡 Fazer o scan em duas etapas (rápido em todas as portas, depois detalhado nas abertas) economiza muito tempo.

---

## 13. Cheat sheet final

```bash
# Básico
nmap <alvo>                              # 1000 portas TCP comuns
nmap -sn <rede>/24                       # descobrir hosts
nmap -Pn <alvo>                          # sem descoberta de host

# Tipos de scan
nmap -sT <alvo>                          # connect (sem root)
sudo nmap -sS <alvo>                     # SYN
sudo nmap -sU <alvo>                     # UDP
sudo nmap -sA <alvo>                     # ACK (firewall)
sudo nmap -sF|-sN|-sX <alvo>             # FIN / NULL / Xmas

# Portas
nmap -p22,80,443 <alvo>
nmap -p- <alvo>                          # todas as 65535
nmap -F <alvo>                           # top 100
nmap --top-ports 20 <alvo>
nmap --open <alvo>                       # só abertas

# Informações
nmap -sV <alvo>                          # versões
sudo nmap -O <alvo>                      # SO
sudo nmap -A <alvo>                      # tudo (agressivo)
nmap -sC <alvo>                          # scripts default
nmap --script=vuln <alvo>

# Velocidade
nmap -T4 <alvo>
nmap --min-rate 1000 <alvo>

# Saída
nmap -oA resultado <alvo>
nmap -v <alvo>
nmap --reason <alvo>
```

---

## 14. Erros comuns

- ❌ Achar que `closed` = host offline → o host **respondeu**.
- ❌ Achar que sem resposta ao ping = host morto → use `-Pn`.
- ❌ Esquecer `-p-` e perder serviços em portas altas (o padrão só olha 1.000).
- ❌ Usar `-sS`/`-O` sem `sudo` → o Nmap cai para `-sT` ou falha.
- ❌ Confiar cegamente em `-sF/-sN/-sX` contra Windows.
- ❌ Usar `-T5` e perder portas por pacotes descartados.
- ❌ Não salvar a saída (`-oA`).
- ❌ Escanear alvos sem autorização.

---

## 📚 Referências

- [Documentação oficial do Nmap](https://nmap.org/book/man.html)
- [Nmap Reference Guide](https://nmap.org/book/toc.html)
- [TryHackMe – Nmap Rooms](https://tryhackme.com/)
- [NSE Script Library](https://nmap.org/nsedoc/)
