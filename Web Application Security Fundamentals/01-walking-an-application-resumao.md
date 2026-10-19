# 🕵️ Resumão Developer Tools para Pentest Web

> Anotações de estudo (TryHackMe + complementos) para o repositório de Cybersecurity.
>
> ⚠️ **Use apenas em laboratórios (TryHackMe, HackTheBox, VMs próprias) ou em sistemas para os quais você tenha autorização por escrito.**

---

## 📑 Índice

1. [Ideia central da sala](#1-ideia-central-da-sala)
2. [Navegação manual e View Page Source](#2-navegação-manual-e-view-page-source)
3. [Inspector — DOM](#3-inspector--dom)
4. [Inspector — CSS e o falso paywall](#4-inspector--css-e-o-falso-paywall)
5. [Debugger — minificação, ofuscação e breakpoints](#5-debugger--minificação-ofuscação-e-breakpoints)
6. [Network — requisições e AJAX](#6-network--requisições-e-ajax)
7. [Storage — Local/Session Storage, Cookies e flags de segurança](#7-storage--localsession-storage-cookies-e-flags-de-segurança)
8. [Checklist de reconhecimento manual](#8-checklist-de-reconhecimento-manual)
9. [Erros comuns](#9-erros-comuns)

---

## 1. Ideia central da sala

Toda a sala é resolvida **só com o navegador** — sem Burp Suite, sem scanners, sem automação. A lição que amarra tudo:

> **O que roda no navegador do usuário está, por definição, sob controle do usuário.** HTML, CSS, JavaScript, cookies — tudo isso chega até a máquina do cliente antes de qualquer coisa ser "escondida". Um controle de segurança implementado *só* no client-side não é segurança de verdade — é, na melhor das hipóteses, UX. Segurança real precisa ser reforçada no servidor.

Ferramentas cobertas: Page Source → Inspector → Debugger → Network → Storage.

---

## 2. Navegação manual e View Page Source

Antes de qualquer ferramenta, o primeiro passo é **navegar a aplicação como usuário normal** — mapear páginas, links, formulários, funcionalidades.

Em seguida, **Ctrl+U** (ou botão direito → "View Page Source") mostra o **HTML bruto enviado pelo servidor**, antes de qualquer renderização ou manipulação por JS.

O que procurar no código-fonte:
- **Comentários HTML** (`<!-- -->`) — devs às vezes deixam notas, credenciais de teste, TODOs
- **Links escondidos** — `<a>` para páginas não linkadas na navegação visível (admin panels, páginas de debug)
- Estrutura do `<head>` — meta tags, scripts carregados, frameworks usados

📌 **View Page Source ≠ Inspector**: Page Source mostra o HTML *original* da resposta do servidor. O Inspector mostra o DOM *atual*, já processado e possivelmente alterado por JavaScript.

---

## 3. Inspector — DOM

O **Inspector** (Elements no Chrome) mostra o **DOM ao vivo** — a árvore de elementos como ela existe *agora* na página, depois de JS rodar.

Diferente do Page Source, aqui dá pra ver:
- Elementos **inseridos dinamicamente** por JavaScript
- O **estado atual** de cada elemento (atributos, classes aplicadas)
- Mudanças em tempo real conforme a página interage

📌 Elemento existir no DOM **não significa** que ele está visível na tela — isso é decidido pelo CSS (próxima seção).

---

## 4. Inspector — CSS e o falso paywall

### O caso clássico: paywall só de CSS

Conteúdo "bloqueado" (ex: parte de um artigo, atrás de um botão "assine para continuar lendo") pode já estar **presente no DOM**, só escondido visualmente:

```css
.paywalled-content {
    display: none;
}
```

ou

```css
.paywalled-content {
    filter: blur(5px);
}
```

### Como contornar (via Inspector, sem editar nada no servidor)

1. Selecionar o elemento escondido no painel **Elements/Inspector**
2. No painel de estilos (**Styles**/**Rules**), encontrar a regra que esconde (`display: none`, `visibility: hidden`, `filter: blur(...)`)
3. Desmarcar a regra (clicar no checkbox) ou editar o valor diretamente ali

Essas mudanças são **só locais** (no seu navegador) — não afetam o servidor nem outros usuários, mas bastam pra revelar o conteúdo que já tinha sido entregue.

> **Lição:** se o conteúdo "protegido" já chegou no HTML/DOM do seu navegador, a proteção é só cosmética. O controle de acesso de verdade tem que impedir o dado de sair do servidor, não só escondê-lo depois que já chegou.

---

## 5. Debugger — minificação, ofuscação e breakpoints

No Firefox/Safari chama **Debugger**; no Chrome chama **Sources**. Foca em **JavaScript** (não DOM/CSS).

### Minificação vs. Ofuscação

| Técnica | O que faz | Por quê | Pretty Print resolve? |
|---|---|---|---|
| **Minificação** | Remove espaços, quebras de linha, comentários | Reduzir tamanho do arquivo (performance) | ✅ sim — reintroduz formatação |
| **Ofuscação** | Troca nomes de variáveis/funções por algo sem sentido (`_0x3801f3`) | Dificultar leitura/cópia do código | ❌ não — só reformata, não traduz nomes |

```js
// minificado + ofuscado
const _0x477d=['#F00','textAlign','width','height','BODY', ...];
```

O botão **Pretty Print** (`{ }`) só desfaz a minificação — o código ofuscado continua difícil de ler mesmo formatado.

### Breakpoints

Um **breakpoint** marca uma linha onde a execução do JS **para antes de rodar**. Clicar no número da linha no painel de código insere o ponto de parada (fica destacado, geralmente em azul).

```text
Sem breakpoint:  JS roda normalmente até o fim
Com breakpoint:  JS roda até a linha marcada → PARA → nada depois dela executa
```

### Exemplo prático: elemento que "pisca" e some

```js
flash['remove']();   // notação de colchete = equivalente a flash.remove()
```

Se essa linha remove um elemento da tela logo após ele aparecer (rápido demais pra ler), colocar um breakpoint nela **impede a remoção** — o elemento fica travado na tela.

> **Por que isso importa:** breakpoints permitem interromper **qualquer lógica client-side** antes que ela rode — validações de formulário só em JS, redirecionamentos condicionais, checagens de "usuário é admin?" feitas no front-end. Nada disso é proteção real se pode ser pausado no navegador do próprio usuário.

---

## 6. Network — requisições e AJAX

A aba **Network** mostra **todo tráfego HTTP** entre navegador e servidor — cada requisição, com headers, cookies, corpo enviado e resposta recebida.

### Fluxo básico

1. Abrir a aba Network
2. Recarregar a página → lista todos os recursos carregados (CSS, JS, imagens, etc.)
3. 🗑️ Ícone de lixeira limpa a lista quando fica poluída

### AJAX

**AJAX** = enviar/receber dados em segundo plano, sem recarregar a página. Ao submeter um formulário via AJAX, surge uma nova entrada na lista do Network (ex: `Initiator: jquery.min.js`), mesmo sem nenhuma mudança visual aparente.

### Sub-abas de cada requisição

| Sub-aba | O que mostra |
|---|---|
| **Headers** | Metadados — status, content-type, cookies enviados |
| **Request** | O que **você** enviou (payload/dados do formulário) |
| **Response** | O que o **servidor** devolveu — onde costuma estar a informação relevante |
| **Timings** | Tempo de cada etapa da requisição |
| **Stack Trace** | De onde no código a requisição foi disparada |

📌 Analisar `Request` + `Response` de uma chamada AJAX revela a estrutura real da API por trás do formulário — útil pra testar manipulação de parâmetros depois.

---

## 7. Storage — Local/Session Storage, Cookies e flags de segurança

A aba **Storage** (ou **Application** no Chrome) mostra o que fica **guardado no navegador** entre requisições.

| Tipo | Persistência | Uso típico |
|---|---|---|
| **Local Storage** | Permanente (sobrevive a fechar o navegador) | Preferências, tokens de app, cache de dados |
| **Session Storage** | Só dura a aba/sessão atual | Estado temporário de formulários, navegação |
| **Cookies** | Definido pelo servidor (tem data de expiração) | Sessão, autenticação, tracking |
| **Cache Storage** | Recursos cacheados (imagens, scripts, respostas de API) | Performance / offline |

### Cookies e suas flags de segurança

Exemplo de cookie de sessão mal configurado:

```
Name: session
Value: 358a9e894eae26388da550b3d95d4b47
HttpOnly: False
Secure: False
SameSite: (vazio)
```

| Flag | Protege contra | Se ausente |
|---|---|---|
| **HttpOnly** | Acesso via JavaScript (`document.cookie`) | Um XSS na aplicação permite **roubar o cookie** direto pelo JS |
| **Secure** | Envio em conexões HTTP não criptografadas | Em HTTP puro, o cookie trafega em **texto plano** — capturável via sniffing/MITM |
| **SameSite** | Envio em requisições cross-site | Abre espaço pra **CSRF** — o navegador manda o cookie mesmo em requisição disparada por outro site |

> **Lição:** um cookie de sessão sem essas três flags é, na prática, uma sessão pronta pra ser sequestrada por três caminhos diferentes (XSS, MITM, CSRF). As flags existem justamente pra fechar essas portas.

💡 Em pentest real, vale também inspecionar Local/Session Storage — tokens, preferências e até credenciais mal guardadas aparecem ali com frequência.

---

## 8. Checklist de reconhecimento manual

```text
1) Navegar a aplicação como usuário comum — mapear páginas e funcionalidades
2) Ctrl+U → ler o HTML bruto: comentários, links escondidos, meta tags
3) Inspector (Elements) → conferir o DOM atual; procurar conteúdo "escondido" só por CSS
4) Inspector (Styles) → testar remover/alterar regras CSS que escondem conteúdo
5) Debugger (Sources) → Pretty Print nos .js minificados; procurar lógica de validação/remoção
6) Debugger → colocar breakpoints em funções suspeitas (remove(), redirect, checks de admin)
7) Network → recarregar com a aba aberta; observar recursos carregados
8) Network → interagir com formulários/botões; inspecionar chamadas AJAX (Request + Response)
9) Storage → Cookies: nome, valor, HttpOnly/Secure/SameSite
10) Storage → Local Storage e Session Storage: tokens, preferências, dados sensíveis
```

---

## 9. Erros comuns

- ❌ Confiar que `display:none` ou `.remove()` client-side é controle de acesso real.
- ❌ Parar no Pretty Print achando que "desofuscou" o código — ele só reformata, não traduz nomes.
- ❌ Ignorar a aba **Response** de uma chamada AJAX e olhar só os Headers.
- ❌ Não checar as flags `HttpOnly`/`Secure`/`SameSite` de cookies de sessão.
- ❌ Pular o View Page Source achando que o Inspector já mostra tudo (Page Source é o HTML original do servidor; Inspector é o DOM já processado).
- ❌ Usar ferramentas automatizadas antes de entender manualmente o que a aplicação está fazendo.

---

## 📚 Referências

- [TryHackMe – Browser-Based Pentesting / DevTools Rooms](https://tryhackme.com/)
- [MDN – Firefox Developer Tools](https://developer.mozilla.org/en-US/docs/Tools)
- [Chrome DevTools Documentation](https://developer.chrome.com/docs/devtools/)
- [OWASP – Client-Side Testing](https://owasp.org/www-project-web-security-testing-guide/)
