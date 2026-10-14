Resumão Nmap — Anotações TryHackMe
Este é o resumo dos principais conceitos que estudamos: estados de portas, flags TCP, descoberta de hosts, tipos de scan e ajustes de velocidade. Organizei em formato de anotação para você salvar no seu repositório de estudos de Cybersecurity.
1. O que é o Nmap?
O Nmap (Network Mapper) é uma ferramenta usada para descobrir hosts ativos em uma rede, identificar portas abertas e, dependendo das opções utilizadas, detectar serviços e suas versões.
Exemplo básico:
nmap 10.10.10.10


Por padrão, o Nmap verifica as 1.000 portas TCP mais comuns.
2. Estados das portas
Os seis estados que você precisa conhecer:
| Estado | Significado |
|---|---|
| open | Existe um serviço aceitando conexões naquela porta. |
| closed | A porta responde, mas não há serviço aceitando conexões nela. |
| filtered | Um firewall ou outro filtro impede determinar se está aberta ou fechada. |
| unfiltered | A porta é acessível à sondagem, mas não foi possível determinar se está aberta. |
| open|filtered | Não foi possível distinguir entre porta aberta e filtrada. |
| closed|filtered | Não foi possível distinguir entre porta fechada e filtrada. |
Importante: uma porta closed não significa que o computador está desligado. Da mesma forma, não receber resposta não significa necessariamente que o host está offline.
3. Flags TCP
As flags são indicadores presentes nos pacotes TCP que controlam ou informam o estado de uma conexão.
Flag	Função
SYN	Inicia uma conexão TCP.
ACK	Confirma o recebimento de dados ou a etapa de conexão.
RST	Reinicia ou interrompe uma conexão.
FIN	Indica que o emissor terminou de enviar dados.
PSH	Solicita que os dados recebidos sejam entregues prontamente à aplicação.
URG	Indica que o campo de ponteiro de urgência é relevante.
TCP 3-way handshake
É o processo normal de estabelecimento de uma conexão TCP:
1. SYN
Cliente solicita conexão.

2. SYN/ACK
Servidor responde.

3. ACK
Cliente confirma; conexão estabelecida.


4. Tipos de varredura do Nmap
Opção	Nome	Para que serve
-sT	TCP Connect Scan	Completa o handshake TCP para verificar portas.
-sS	SYN Scan	Usa SYN para verificar portas sem completar o handshake normalmente.
-sA	ACK Scan	Ajuda a analisar filtragem de portas e regras de firewall; não determina diretamente se uma porta está aberta.
-sF	FIN Scan	Envia pacotes com a flag FIN.
-sN	NULL Scan	Envia pacotes sem flags TCP definidas.
-sX	Xmas Scan	Envia pacotes com FIN, PSH e URG.
Diferença essencial: -sT vs. -sS
-sT — conexão completa
SYN → SYN/ACK → ACK
O handshake é completado. O Nmap normalmente encerra a conexão em seguida.

-sS — SYN Scan
SYN → SYN/ACK → RST
Se a porta estiver aberta, o Nmap interrompe a tentativa sem enviar o ACK final do handshake.


O -sT normalmente não exige privilégios elevados. O -sS geralmente exige root ou sudo para enviar pacotes brutos.
Não confunda: -sS verifica portas TCP; não é a mesma coisa que dar ping.
5. Descoberta de hosts ativos
O objetivo aqui é descobrir quais computadores respondem na rede, sem necessariamente verificar todas as portas.
Opção	Função
-sn	Faz descoberta de hosts sem realizar a varredura de portas.
-PR	Usa sondagens ARP para descoberta de hosts em redes Ethernet locais compatíveis.
-PS	Envia sondagens TCP SYN para descoberta de hosts.
-PA	Envia sondagens TCP ACK para descoberta de hosts.
-PU	Envia sondagens UDP para descoberta de hosts.
--reason	Mostra o motivo pelo qual o Nmap classificou um host ou uma porta daquele modo.
Exemplos:
nmap -sn 10.10.10.0/24
nmap -sn --reason 10.10.10.0/24
nmap -PS -sn 10.10.10.0/24
sudo nmap -PU -sn 10.10.10.0/24


Ping vs. SYN Scan
- ping: normalmente usa ICMP para verificar se o host responde.
- -sS: verifica portas TCP específicas.
- -PS: usa sondagens TCP SYN para tentar descobrir hosts ativos.
Um host pode bloquear ping e ainda ter portas TCP abertas. Por isso, não responder ao ping não prova que o host está offline.
6. Escolhendo as portas
Estas opções controlam quais portas serão verificadas.
Opção	O que faz
-p22,80,443	Verifica somente as portas 22, 80 e 443.
-p20-25	Verifica as portas de 20 até 25, inclusive.
-p1-1023	Verifica as portas de 1 até 1023.
-p-	Verifica todas as 65.535 portas TCP, de 1 a 65.535.
-F	Verifica as 100 portas mais comuns.
--top-ports 10	Verifica as 10 portas mais comuns.
-r	Examina as portas em ordem numérica, em vez da ordem aleatória padrão.
Exemplos:
nmap -sS -p22,80,443 10.10.10.10
nmap -sS -p- 10.10.10.10
nmap -sT -F 10.10.10.10
nmap --top-ports 10 10.10.10.10


Portas comuns para memorizar
Porta	Serviço normalmente associado
21	FTP
22	SSH
25	SMTP
53	DNS
80	HTTP
110	POP3
143	IMAP
443	HTTPS
445	SMB
3306	MySQL
3389	RDP
5432	PostgreSQL
Essas associações são comuns, mas um serviço pode ser configurado para usar outra porta.
7. Velocidade da varredura
O parâmetro -T controla o perfil de temporização.
Opção	Perfil	Característica
-T0	Paranoid	Extremamente lento.
-T1	Sneaky	Muito lento.
-T2	Polite	Mais cuidadoso.
-T3	Normal	Padrão do Nmap.
-T4	Aggressive	Mais rápido; comum em laboratórios e CTFs.
-T5	Insane	Muito agressivo; pode prejudicar a precisão.
Exemplo:
nmap -sS -T4 10.10.10.10


Um perfil mais lento não garante invisibilidade. Firewalls e sistemas de monitoramento podem detectar as sondagens independentemente da velocidade.
Taxa de envio e paralelização
Opção	Função
--min-rate 100	Solicita uma taxa mínima de envio de aproximadamente 100 pacotes por segundo, quando possível.
--max-rate 10	Limita a taxa de envio a aproximadamente 10 pacotes por segundo.
--min-parallelism 10	Solicita pelo menos 10 sondagens simultâneas, quando possível.
--max-parallelism 20	Limita a paralelização a 20 sondagens simultâneas.
Diferença importante:
- Rate: quantidade de pacotes enviados por segundo.
- Parallelism: quantidade de sondagens executadas simultaneamente.
8. Serviços e versões
Depois de encontrar portas abertas, você pode tentar identificar os serviços e suas versões:
nmap -sV 10.10.10.10


Para combinar descoberta de hosts, detecção de versões e varredura de portas, escolha as opções conforme o objetivo do teste. Não é necessário usar todas em toda varredura.

9. Exemplos práticos para guardar
Use apenas em laboratórios ou sistemas para os quais você tenha autorização.
Varredura básica
nmap 10.10.10.10


Verifica as portas TCP mais comuns.

SYN Scan nas portas mais comuns
sudo nmap -sS 10.10.10.10


Usa SYN Scan nas portas padrão.

Verificar todas as portas TCP
sudo nmap -sS -p- 10.10.10.10


Examina todas as portas TCP.

Verificar versões dos serviços
nmap -sV 10.10.10.10


Tenta identificar os serviços e suas versões.

Descobrir hosts ativos em uma sub-rede
nmap -sn 10.10.10.0/24


Faz descoberta de hosts sem varredura de portas.

Varredura mais rápida em laboratório
sudo nmap -sS -p- -T4 10.10.10.10


Combina SYN Scan, todas as portas TCP e perfil de temporização T4.
