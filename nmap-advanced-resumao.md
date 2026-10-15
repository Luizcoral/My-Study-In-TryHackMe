# 🛰️ Resumão Nmap Advanced — Anotações TryHackMe

> Continuação do Nmap Basic Port Scans. Aqui entram os scans "não-SYN", spoofing, decoys, idle scan, fragmentação e as opções de detalhamento.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [Null, FIN e Xmas Scan](#1-null-fin-e-xmas-scan)
2. [Maimon Scan](#2-maimon-scan)
3. [ACK, Window e Custom Scan](#3-ack-window-e-custom-scan)
4. [Spoofing e Decoys](#4-spoofing-e-decoys)
5. [Idle / Zombie Scan](#5-idle--zombie-scan)
6. [Pacotes fragmentados](#6-pacotes-fragmentados)
7. [Detalhamento do scan](#7-detalhamento-do-scan)
8. [Tabela-resumo da sala](#8-tabela-resumo-da-sala)
9. [Fluxo de raciocínio para a prova/CTF](#9-fluxo-de-raciocínio-para-a-provactf)
10. [Erros comuns](#10-erros-comuns)

---

## 1. Null, FIN e Xmas Scan

Pacotes TCP "estranhos", sem o SYN que inicia uma conexão normal. Diferem só nas flags enviadas:

| Scan | Opção | Flags enviadas |
|---|---|---|
| Null | `-sN` | Nenhuma (os 6 bits em zero) |
| FIN | `-sF` | Só `FIN` |
| Xmas | `-sX` | `FIN` + `PSH` + `URG` |

> O nome Xmas vem das "luzinhas de Natal": várias flags acesas ao mesmo tempo.

### Lógica (invertida em relação ao SYN Scan)

| Resposta do alvo | Interpretação |
|---|---|
| `RST` | Porta **closed** |
| Nenhuma resposta | Porta **open\|filtered** |

Uma porta aberta ignora esses pacotes (não fazem sentido para uma conexão). Uma fechada responde com `RST`. Por isso o Nmap nunca confirma `open` — só `open|filtered`: o silêncio pode ser porta aberta **ou** firewall descartando o pacote.

```bash
sudo nmap -sN 10.67.180.127
sudo nmap -sF 10.67.180.127
sudo nmap -sX 10.67.180.127
```

### Quando usar

Útil contra **firewalls stateless**, que só bloqueiam pacotes com SYN (detectam tentativa de conexão pela flag). Como esses scans não têm SYN, passam.

Um **firewall stateful** (acompanha o estado das conexões) bloqueia os três do mesmo jeito — nesse caso não adianta usá-los.

| Scan | Discrição | Quando tentar |
|---|---|---|
| FIN | Menos chamativo (FIN existe em conexões normais) | Primeira tentativa |
| Null | Nenhuma flag é incomum | Se FIN for barrado |
| Xmas | Mais chamativo, fácil de ter assinatura em IDS | Estudo/lab, ou se os outros falharem |

### Limitações

- Exigem `sudo` (pacotes brutos).
- **Windows e vários dispositivos de rede não seguem a RFC**: respondem `RST` para tudo, e o resultado fica errado (tudo parece `closed`).
- Confirme sempre as portas `open|filtered` com `-sS -sV`:
  ```bash
  nmap -sS -sV -p<portas> 10.67.180.127
  ```

---

## 2. Maimon Scan

Descrito por Uriel Maimon em 1996. Envia **FIN + ACK** juntos (opção `-sM`).

- **FIN+ACK é uma combinação comum** no encerramento normal de uma conexão TCP, então o pacote parece legítimo.
- A ideia era igual à do Null/FIN/Xmas: `RST` = fechada, silêncio = candidata a aberta.
- Alguns sistemas **BSD antigos** descartavam o pacote quando a porta estava aberta, revelando-a.

```bash
sudo nmap -sM 10.67.180.127
```

⚠️ **Praticamente inútil hoje.** A maioria dos sistemas responde `RST` para tudo, aberta ou fechada, e o Nmap conclui (erradamente) que está tudo fechado. É estudado mais para entender o mecanismo do que para usar na prática.

> **Lição:** um scan dizer "fechada" não prova que está. Sempre que o resultado parecer estranho, confirme com outra técnica.

---

## 3. ACK, Window e Custom Scan

Diferente dos anteriores, esses **não descobrem portas abertas** — servem para **mapear regras de firewall**.

### ACK Scan (`-sA`)

Envia só a flag `ACK`. Como não há conexão em andamento, o alvo responde `RST` **em qualquer porta**, aberta ou fechada.

| Resposta | Estado no Nmap | Significado |
|---|---|---|
| `RST` | `unfiltered` | O pacote **passou** pelo firewall |
| Silêncio / ICMP | `filtered` | O firewall **bloqueou** o pacote |

```bash
sudo nmap -sA 10.67.180.127
```

Sem firewall: tudo `unfiltered` (nenhuma informação útil, como no exemplo do material, onde as 1000 portas vieram `unfiltered`). Com firewall ativo: só as portas liberadas pelas regras respondem — no exemplo, 22, 25, 80, 443 e 5000 ficaram `unfiltered`, e as outras 995, `filtered`.

### Window Scan (`-sW`)

Quase idêntico ao ACK, mas examina o campo **Window** do `RST` de volta. Em alguns sistemas esse valor muda conforme a porta esteja aberta ou fechada, permitindo diferenciá-las — algo que o ACK sozinho não faz.

```bash
sudo nmap -sW 10.67.180.127
```

No exemplo do material, as mesmas 5 portas liberadas pelo firewall (22, 25, 80, 443, 5000) apareceram como `closed` no Window Scan, em vez de `unfiltered` como no ACK.

> `closed` no resultado do Window Scan **não** significa porta fechada de verdade — só que o Linux usa janela zero nesse caso. O que importa é que a porta **respondeu**, ou seja, não está bloqueada pelo firewall.

### Custom Scan (`--scanflags`)

Monta sua própria combinação de flags:

```bash
nmap --scanflags RSTSYNFIN 10.67.180.127
```

Exige entender como o alvo reage a cada combinação para interpretar o resultado corretamente.

### Resumo do bloco

> **ACK e Window revelam as regras do firewall, não os serviços.** Uma porta `unfiltered` não está necessariamente aberta — só não está bloqueada. As regras podem estar desatualizadas e liberar portas sem serviço algum escutando.

---

## 4. Spoofing e Decoys

### IP Spoofing (`-S`)

O Nmap monta pacotes com um **IP de origem falso**.

```bash
nmap -e NET_INTERFACE -Pn -S SPOOFED_IP 10.67.180.127
```

| Opção | Função |
|---|---|
| `-S <ip>` | Define o IP de origem falso |
| `-e <interface>` | Interface de rede a usar |
| `-Pn` | Pula a descoberta de host (você não receberá o ping de volta) |

**Problema:** o alvo responde para o IP falso, não para você. Você precisa **monitorar o tráfego** (ex. Wireshark) para capturar a resposta, senão o scan é inútil. Só funciona em cenários específicos, geralmente na mesma rede local — filtros de provedores (ingress filtering) costumam bloquear IP de origem falso pela internet.

### MAC Spoofing

```bash
nmap --spoof-mac SPOOFED_MAC 10.67.180.127
```

Só funciona na **mesma rede Ethernet/Wi-Fi** do alvo (MAC não atravessa roteadores).

### Decoy Scan (`-D`)

Em vez de falsificar e depender de capturar o tráfego, **mistura seu IP real no meio de vários outros**, dificultando identificar qual é o verdadeiro atacante.

```bash
nmap -D 10.10.0.1,10.10.0.2,ME 10.67.180.127
nmap -D 10.10.0.1,10.10.0.2,RND,RND,ME 10.67.180.127
```

| Termo | Significado |
|---|---|
| IP específico | Isca com IP escolhido |
| `RND` | Isca com IP aleatório (muda a cada execução) |
| `ME` | Seu IP real, na posição indicada |

Diferente do spoofing puro, aqui **você ainda recebe suas próprias respostas** — o scan funciona normalmente, só que o alvo vê várias origens.

| Técnica | Seu IP aparece? | Você recebe respostas? |
|---|---|---|
| Spoofing (`-S`) | Não | Não (precisa monitorar) |
| Decoys (`-D`) | Sim, misturado | Sim |

⚠️ **Decoys não te tornam invisível** — seu IP real continua nos logs, só misturado. Use iscas que estejam online, para não causar efeitos colaterais indesejados (tráfego parecendo SYN flood) em terceiros.

---

## 5. Idle / Zombie Scan

Resolve o problema do spoofing simples: usa uma máquina "zumbi" para descobrir o resultado **sem nunca receber resposta direta do alvo**.

```bash
nmap -sI ZOMBIE_IP 10.67.180.127
```

### Como funciona

Baseia-se no **IP ID**, um contador no cabeçalho IP que sobe +1 a cada pacote enviado (em sistemas simples e previsíveis).

```text
1) Você → Zumbi:  SYN/ACK
   Zumbi → Você:  RST   (IP ID = X)         ← anota o valor

2) Você → Alvo:   SYN com origem FALSA = IP do zumbi

3) Você → Zumbi:  SYN/ACK
   Zumbi → Você:  RST   (IP ID = ?)         ← compara com X
```

No passo 2, o que acontece depende do estado da porta no alvo:

| Porta no alvo | Alvo responde ao zumbi com | Zumbi faz | IP ID |
|---|---|---|---|
| Fechada | `RST` | Ignora | Não sobe |
| Aberta | `SYN/ACK` | Estranha e manda `RST` | Sobe +1 |
| Filtrada | Nada | Nada | Não sobe |

### Interpretando o resultado

| Diferença de IP ID (passo 3 vs passo 1) | Conclusão |
|---|---|
| **1** | Porta fechada ou filtrada (zumbi só respondeu a você) |
| **2** | Porta **aberta** (zumbi também respondeu ao alvo) |

### Pontos-chave

- **Vantagem:** o alvo só vê o IP do zumbi — seu IP real nunca aparece nos logs dele. É o scan mais furtivo da sala.
- **O zumbi precisa estar ocioso** (sem outro tráfego), senão o IP ID sobe por motivos alheios e o scan fica inútil — daí o nome.
- **Candidatos a zumbi:** sistemas com IP ID incremental e previsível — impressoras, câmeras IP, dispositivos IoT, SOs antigos. Linux/Windows modernos randomizam o IP ID e não servem.
- **Limitação:** não distingue fechada de filtrada (ambas dão diferença 1). É lento.
- Costuma precisar de `-Pn`:
  ```bash
  sudo nmap -sI ZOMBIE_IP -Pn -p80,443 10.67.180.127
  ```

---

## 6. Pacotes fragmentados

### Firewall vs IDS

| | Firewall | IDS |
|---|---|---|
| Função | Permite/bloqueia por regras | Detecta e alerta sobre comportamento suspeito |
| Olha | Cabeçalhos IP e transporte | Cabeçalhos **e** o conteúdo (assinaturas) |

### Fragmentação no Nmap

| Opção | Efeito |
|---|---|
| `-f` | Fragmenta os dados IP em blocos de **8 bytes** |
| `-ff` (ou `-f -f`) | Blocos de **16 bytes** |
| `--mtu N` | Define o tamanho do fragmento (sempre múltiplo de 8) |
| `--data-length N` | O oposto: **aumenta** o pacote com N bytes extras, para parecer mais inocente |

```bash
sudo nmap -sS -p80 -f 10.67.180.127
```

### Por que funciona (às vezes)

O cabeçalho TCP do SYN tem 24 bytes. Com `-f`, vira **3 fragmentos IP** (24÷8). Com `-ff`, vira **2 fragmentos** (16 + 8). Cada fragmento carrega:
- **Identification (ID):** mesmo número em todos os fragmentos do mesmo pacote — liga os pedaços.
- **Fragment Offset:** posição de cada pedaço (0, 8, 16...).

Ideia: alguns firewalls/IDS mais simples **não remontam** os fragmentos antes de inspecionar, então não enxergam o pacote completo (ex. a flag SYN pode cair num fragmento separado do resto).

⚠️ **Limitações:**
- Firewalls/IDS modernos remontam os fragmentos antes de inspecionar — a técnica perde o efeito.
- Fragmentos muito pequenos são incomuns no tráfego normal e podem até **levantar suspeita**.
- Alguns alvos/equipamentos descartam fragmentos, piorando o resultado.
- Só funciona com scans de pacote bruto (`-sS`, não `-sT`), e exige `sudo`.

---

## 7. Detalhamento do scan

### `--reason`

Mostra **por que** o Nmap chegou a cada conclusão (coluna REASON):

```bash
sudo nmap -sS --reason 10.67.180.127
```

| Exemplo | Significado |
|---|---|
| `received arp-response` | Host considerado ativo por responder ao ARP |
| `syn-ack ttl 64` | Porta aberta porque respondeu com SYN/ACK (TTL 64 sugere Linux) |
| `994 resets` | 994 portas responderam RST (fechadas) |

### Verbosidade e debug

| Opção | O que faz |
|---|---|
| `-v` | Verboso — mostra portas descobertas em tempo real, sem esperar o fim |
| `-vv` | Mais verboso — mostra fases do scan (ARP, DNS, SYN Scan) e estatísticas (`Raw packets sent/Rcvd`) |
| `-d` | Debug (saída extensa) |
| `-dd` | Debug ainda mais detalhado |

```bash
sudo nmap -sS -v --reason 10.67.180.127
```

💡 Combine `--reason` com `-v` para estudar: ajuda a entender exatamente por que cada porta ficou naquele estado — útil para diagnosticar falsos resultados (como os do Maimon Scan).

### Outras opções úteis citadas na sala

| Opção | Função |
|---|---|
| `--source-port PORT_NUM` | Define a porta de origem do scan |
| `--data-length NUM` | Acrescenta dados extras ao pacote, até o tamanho informado |

---

## 8. Tabela-resumo da sala

| Técnica | Comando | Objetivo |
|---|---|---|
| TCP Null Scan | `sudo nmap -sN <alvo>` | Achar candidatas a aberta via silêncio |
| TCP FIN Scan | `sudo nmap -sF <alvo>` | Idem, flag menos chamativa |
| TCP Xmas Scan | `sudo nmap -sX <alvo>` | Idem, flags combinadas |
| TCP Maimon Scan | `sudo nmap -sM <alvo>` | Idem (só funciona em BSD antigo) |
| TCP ACK Scan | `sudo nmap -sA <alvo>` | Mapear regras de firewall |
| TCP Window Scan | `sudo nmap -sW <alvo>` | Mapear firewall + distinguir estado em alguns SOs |
| Custom Scan | `sudo nmap --scanflags URGACKPSHRSTSYNFIN <alvo>` | Combinação própria de flags |
| Spoofed Source IP | `sudo nmap -S SPOOFED_IP <alvo>` | Esconder origem (exige monitorar o tráfego) |
| Spoofed MAC | `--spoof-mac SPOOFED_MAC` | Falsificar MAC (rede local) |
| Decoy Scan | `nmap -D DECOY_IP,ME <alvo>` | Misturar origem real com iscas |
| Idle/Zombie Scan | `sudo nmap -sI ZOMBIE_IP <alvo>` | Scan via intermediário, sem expor seu IP |
| Fragmentar em 8 bytes | `-f` | Driblar inspeção simples de firewall/IDS |
| Fragmentar em 16 bytes | `-ff` | Idem, fragmentos maiores |
| Porta de origem | `--source-port PORT_NUM` | Definir a porta de origem do scan |
| Dados extras | `--data-length NUM` | Aumentar o pacote (parecer mais "normal") |
| Explicar conclusões | `--reason` | Mostrar o porquê de cada resultado |
| Verbosidade | `-v` / `-vv` | Mais detalhes durante o scan |
| Debug | `-d` / `-dd` | Diagnóstico aprofundado |

### Como cada família de scan provoca resposta

> **Null, FIN e Xmas** provocam resposta de portas **fechadas** (RST). Silêncio = candidata a aberta.
> **Maimon, ACK e Window** provocam resposta de portas **abertas e fechadas** (ambas mandam RST) — por isso servem para outra coisa (mapear firewall), não para achar portas abertas.

---

## 9. Fluxo de raciocínio para a prova/CTF

```text
1. Quero achar portas candidatas sem SYN?
   → Null / FIN / Xmas (RST = fechada, silêncio = candidata)

2. Suspeito de firewall e quero saber o que ele deixa passar?
   → ACK (unfiltered/filtered) e Window (pode distinguir estado)

3. Quero confirmar de verdade o que está aberto?
   → -sS -sV (resposta direta, sem ambiguidade)

4. Quero esconder minha origem?
   → Tenho como monitorar o tráfego de volta?
       Sim → Spoofing (-S)
       Não → Decoy (-D) ou Idle Scan (-sI, o mais furtivo)

5. Suspeito de firewall/IDS simples que não remonta pacotes?
   → Fragmentação (-f / -ff)

6. Quero entender por que o Nmap concluiu algo?
   → --reason (+ -v/-vv para acompanhar em tempo real)
```

---

## 10. Erros comuns

- ❌ Achar que `open|filtered` confirma porta aberta — é só "não fechada".
- ❌ Usar Null/FIN/Xmas/Maimon contra Windows e confiar no resultado.
- ❌ Achar que o ACK/Window Scan revela serviços — eles revelam **regras de firewall**.
- ❌ Usar `-S` sem conseguir capturar o tráfego de resposta.
- ❌ Achar que Decoy Scan esconde completamente o IP real — ele só mistura.
- ❌ Escolher um zumbi "ocupado" para o Idle Scan — o resultado vira lixo.
- ❌ Confiar em fragmentação contra firewalls/IDS modernos, que remontam os pacotes.
- ❌ Esquecer `-Pn` quando o spoofing/idle scan não recebe resposta de ping.

---

## 📚 Referências

- [Documentação oficial do Nmap](https://nmap.org/book/man.html)
- [Nmap Reference Guide](https://nmap.org/book/toc.html)
- [TryHackMe – Nmap Advanced Port Scans](https://tryhackme.com/)
