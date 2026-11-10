# Web Hacking — TryHackMe | Relatório técnico

**Trilha:** Cyber Security 101  
**Módulo:** Web Hacking  
**Plataforma:** TryHackMe  
**Período:** 08–10/11/2026  
**Ambiente:** TryHackMe AttackBox, navegador Firefox e aplicações didáticas autorizadas  
**Objetivo profissional:** SOC Analyst Jr. / Blue Team  
**Status:** módulo concluído, conforme informado pelo estudante.

> **Ética, evidências e privacidade:** este relatório reúne os registros da conversa principal e de outra conversa em que parte dos exercícios foi estudada. Apenas ações com evidências disponíveis são apresentadas como execução confirmada. Senhas, flags, tokens de autenticação e respostas de desafios foram intencionalmente omitidos. Endereços IP do laboratório são temporários e não servem como alvos permanentes.

## Sumário

1. [Objetivos e panorama](#1-objetivos-e-panorama)
2. [Web Application Basics](#2-web-application-basics)
3. [JavaScript Essentials](#3-javascript-essentials)
4. [SQL Fundamentals](#4-sql-fundamentals)
5. [Burp Suite: The Basics](#5-burp-suite-the-basics)
6. [Laboratório de proxy, escopo e site map](#6-laboratório-de-proxy-escopo-e-site-map)
7. [Demonstração autorizada de XSS](#7-demonstração-autorizada-de-xss)
8. [Dificuldades, correções e decisões técnicas](#8-dificuldades-correções-e-decisões-técnicas)
9. [Glossário de comandos, métodos e siglas](#9-glossário-de-comandos-métodos-e-siglas)
10. [Aplicações para SOC Nível 1](#10-aplicações-para-soc-nível-1)
11. [Evidências, limitações e conclusão](#11-evidências-limitações-e-conclusão)

---

## 1. Objetivos e panorama

O módulo apresentou os fundamentos de comunicação web, linguagens do navegador, bancos de dados e inspeção de requisições HTTP/HTTPS. O objetivo prático foi compreender como aplicações funcionam, como testes autorizados revelam comportamentos inseguros e como essas evidências ajudam uma equipe defensiva.

| Sala | Tópicos principais | Registro disponível |
|---|---|---|
| **Web Application Basics** | Front-end/back-end, URL, HTTP, headers, cookies, códigos de status, cabeçalhos de segurança, API | Exercícios e respostas confirmados ao longo das Tasks 1–10; prática GET e POST observada |
| **JavaScript Essentials** | Variáveis, console, scripts internos/externos, DOM, diálogos, validação de cliente e minificação | Histórico recuperado de outra conversa; saídas específicas disponíveis em parte |
| **SQL Fundamentals** | Bancos relacionais, MySQL, SQL, CRUD, cláusulas, operadores e funções | Acesso MySQL e prática CRUD documentados; parte das consultas posteriores aparece apenas como exercício explicado |
| **Burp Suite: The Basics** | Proxy, FoxyProxy, Site map, Scope, interceptação HTTPS, edição de requisições e XSS | Prints e saídas práticas confirmados; conclusão informada pelo estudante |

**Separação de funções:** um analista SOC N1 normalmente investiga logs e alertas e escala incidentes; o Burp Suite é principalmente uma ferramenta de teste de segurança web. Conhecer o Burp ajuda a compreender o tráfego e as técnicas que podem aparecer em alertas, mas não autoriza testes ativos em sites de terceiros.

## 2. Web Application Basics

### 2.1 Arquitetura de aplicações web

- **Front-end:** código executado ou renderizado no navegador. HTML organiza estrutura, CSS define apresentação, JavaScript viabiliza interatividade.
- **Back-end:** processamento de requisições, regras de negócio, acesso a bancos de dados e integração com outros serviços.
- **Web server:** recebe e responde requisições HTTP/HTTPS.
- **Database:** armazena e disponibiliza dados conforme permissões e lógica da aplicação.
- **WAF (Web Application Firewall):** examina características do tráfego web e pode bloquear padrões considerados maliciosos.

### 2.2 Anatomia da URL

Exemplo didático:

```text
https://exemplo.com/produtos/1?idioma=pt#detalhes
```

| Componente | Exemplo | Significado |
|---|---|---|
| Scheme / protocolo | `https` | Forma de acesso; HTTPS usa TLS |
| Host / domínio | `exemplo.com` | Servidor ou domínio de destino |
| Path | `/produtos/1` | Caminho do recurso |
| Query string | `?idioma=pt` | Parâmetros enviados na URL |
| Fragment | `#detalhes` | Referência local a parte do documento; geralmente não é enviada ao servidor HTTP |

Também foi estudado **typosquatting**: registrar endereços muito parecidos com domínios legítimos para aproveitar erros de digitação e enganar usuários.

### 2.3 Estrutura de mensagens HTTP

Uma requisição HTTP/1.1 possui linha inicial, cabeçalhos, uma linha vazia e corpo opcional:

```http
POST /api/user/2 HTTP/1.1
Host: exemplo.com
Content-Type: application/x-www-form-urlencoded

country=US
```

A resposta HTTP/1.1 tem linha de status, cabeçalhos, linha vazia e corpo opcional:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"ok"}
```

**Métodos estudados:** `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS`, `TRACE` e `CONNECT`. `OPTIONS` pode indicar métodos aceitos por um recurso; `GET` solicita dados; `POST` envia dados; `DELETE` solicita exclusão.

**Códigos de resposta:**

| Família | Interpretação | Exemplo |
|---|---|---|
| `1xx` | Informação provisória | `100 Continue` |
| `2xx` | Sucesso | `200 OK` |
| `3xx` | Redirecionamento | `301 Moved Permanently` |
| `4xx` | Erro na requisição/cliente | `404 Not Found` |
| `5xx` | Falha do lado do servidor/intermediário | `500 Internal Server Error` |

**Headers examinados:** `Host`, `User-Agent`, `Referer`, `Cookie`, `Content-Type`, `Content-Length`, `Server`, `Set-Cookie` e `Location`.

### 2.4 Cookies e cabeçalhos de segurança

- `Secure` restringe o envio do cookie a conexões seguras.
- `HttpOnly` impede acesso ao cookie por APIs de JavaScript como `document.cookie`, mas não elimina XSS.
- `Content-Security-Policy` e a diretiva `script-src` restringem origens permitidas para scripts.
- `Strict-Transport-Security` com `includeSubDomains` estende a política HTTPS aos subdomínios.
- `X-Content-Type-Options: nosniff` reduz certos riscos de interpretação indevida de tipo MIME.
- `Referrer-Policy` controla quanto do endereço de origem pode ser enviado no cabeçalho `Referer`.

### 2.5 Prática da API didática

Na Task 10, a aplicação simulou requisições HTTP a uma API:

| Operação | Endpoint | Objetivo | Evidência |
|---|---|---|---|
| `GET` | `/api/users` | Recuperar lista de usuários | Resposta `200 OK` e tabela HTML com Alice, Bob e Charlie; desafio validado |
| `POST` | `/api/user/2` | Alterar país de Bob de `UK` para `US` | Corpo `country=US` observado, `Content-Length: 9`; segunda pergunta confirmada pelo estudante |
| `DELETE` | `/api/user/2` | Solicitar exclusão de Bob | Método e destino configurados e envio orientado; não há saída da resposta final preservada |

**Lição:** o método HTTP, o caminho, os cabeçalhos e o corpo desempenham funções diferentes. Uma requisição pode estar correta na sintaxe, mas seu processamento depende das validações e da lógica do servidor.

## 3. JavaScript Essentials

> Conteúdo recuperado de outra conversa. Há prática confirmada para alguns itens, mas não há registro completo de todas as saídas nem necessidade de repetir respostas de desafios.

### 3.1 Variáveis, operações e console

Exercício didático com `let`, soma e impressão:

```javascript
let x = 5;
let y = 10;
let result = x + y;
console.log("The result is: " + result);
```

Resultado estudado: `The result is: 15`. Uma variação alterando `x` para 10 produziu o resultado esperado `20`.

**Conceito:** `undefined` após certas declarações no console não significa necessariamente erro; pode apenas indicar que a expressão avaliada não retorna um valor exibível.

### 3.2 JavaScript interno, externo e DOM

- **Interno:** código dentro de `<script>...</script>` no HTML.
- **Externo:** código referenciado com `<script src="script.js"></script>`.
- **DOM (Document Object Model):** representação do documento que permite acessar e alterar elementos da página.

Foi trabalhada a instrução:

```javascript
document.getElementById("result").innerHTML = "The result is: 15";
```

**Resultado observado:** a página exibiu `Addition of Two Numbers` e `The result is: 15`.

**Cuidado defensivo:** `innerHTML` pode interpretar conteúdo como HTML; inserir dados não confiáveis nesse local sem sanitização/contexto apropriado pode introduzir XSS.

### 3.3 Funções de diálogo

Exemplos estudados:

```javascript
alert("HelloTHM");
let name = prompt("What is your name?");
alert("Hello " + name);
let confirmed = confirm("Are you sure?");
```

- `alert`: exibe aviso.
- `prompt`: solicita uma entrada textual.
- `confirm`: retorna um valor booleano (`true` ao confirmar e `false` ao cancelar).

### 3.4 Fluxo de controle, minificação e boas práticas

A sala também abordou condicionais e a possibilidade de contornar validações executadas somente no cliente. Na atividade de código minificado, foram discutidos o alerta `Welcome to THM` e o valor de idade `21` identificados durante a análise.

**Lições:** minificar não equivale a proteger segredos; código JavaScript entregue ao navegador pode ser inspecionado. Autorização, validação de dados e decisões sensíveis precisam ser verificadas no servidor. Não devem existir senhas ou segredos embutidos em JavaScript público.

## 4. SQL Fundamentals

> Conteúdo recuperado de outra conversa, distinguindo comandos com execução observada de exemplos apresentados para entendimento.

### 4.1 Modelo de dados

- **Banco relacional:** organiza dados em tabelas com linhas e colunas.
- **Chave primária (Primary Key):** identifica uma linha de forma única.
- **Chave estrangeira (Foreign Key):** referencia dados de outra tabela, mantendo relacionamentos.
- **DBMS (Database Management System):** sistema gerenciador de banco de dados.
- **SQL (Structured Query Language):** linguagem usada para consultar e manipular bancos relacionais.

### 4.2 MySQL e navegação pelo banco

O laboratório trabalhou a conexão ao cliente MySQL e estes comandos:

```sql
SHOW DATABASES;
USE task_4_db;
SHOW TABLES;
```

No ambiente de teste, também foi criado um banco de prática denominado `pratica_crud`, com uma tabela `books` para estudar operações de manipulação.

### 4.3 CRUD — prática observada

CRUD significa **Create, Read, Update, Delete**. O histórico registra:

1. `INSERT` de um registro de livro didático sobre segurança Android.
2. `SELECT` que confirmou o registro (ID 1).
3. `UPDATE` de um campo de descrição, conferido posteriormente com o valor `Architectures`.
4. `DELETE` do registro e verificação final do estado da tabela.

Esquema ilustrativo da sintaxe, **não transcrição literal de todas as colunas do laboratório**:

```sql
SELECT * FROM books;
UPDATE books SET description = 'Architectures' WHERE id = 1;
DELETE FROM books WHERE id = 1;
```

**Ocorrências técnicas importantes:** um erro SQL `1064` surgiu de instrução incompleta ou sintaxe incorreta; o erro `1193` apareceu após uso isolado de `SET`. A correção foi utilizar `UPDATE ... SET ... WHERE ...` como uma instrução completa.

### 4.4 Cláusulas, operadores e funções

Nas tarefas seguintes foram tratados conceitos e consultas demonstrativas, sem resultados terminais preservados para cada uma:

| Recurso | Finalidade |
|---|---|
| `WHERE` | Filtrar linhas |
| `ORDER BY` | Ordenar registros |
| `DISTINCT` | Eliminar duplicações no conjunto de resultado |
| `LIKE` | Procurar padrões textuais |
| `>=` | Comparar números |
| `LENGTH()` | Medir comprimento de string |
| `SUM()` | Somar valores numéricos |
| `GROUP_CONCAT()` | Concatenar valores de linhas agrupadas no MySQL |

**Lição de segurança:** consultas com parâmetros e contas com privilégios mínimos ajudam a reduzir riscos. Montar SQL concatenando entradas não confiáveis pode produzir SQL Injection; a correção não é simplesmente ocultar uma mensagem de erro.

## 5. Burp Suite: The Basics

### 5.1 O Burp como proxy de inspeção

Fluxo:

```text
Firefox → FoxyProxy → Burp Proxy → aplicação de laboratório
                         ← respostas ←
```

Um **proxy** é um intermediário de comunicação. Dependendo da ferramenta e da configuração, pode encaminhar, registrar, filtrar ou permitir inspeção do tráfego. O Burp Suite foi utilizado para analisar e modificar requisições HTTP/HTTPS em um laboratório autorizado.

**Edições discutidas:** Community (gratuita e voltada a testes manuais), Professional (recursos avançados de teste), Enterprise (análise automatizada e recorrente em ambientes organizacionais).

**Módulos:** `Proxy`, `Repeater`, `Intruder`, `Decoder`, `Comparer`, `Sequencer`; também foram estudados `Dashboard`, `Target`, `Site map`, `HTTP history`, `Event log` e `Extensions`.

### 5.2 Inicialização na AttackBox

- Burp Suite Community Edition **2026.1.5**, aberto por **Applications → Web → Burp Suite Community Edition**.
- Projeto selecionado: **Temporary project in memory**.
- Configuração selecionada: **Use Burp defaults**.
- Listener local confirmado no Burp: **`127.0.0.1:8080`**.
- Firefox com extensão **FoxyProxy**, perfil pré-configurado **Burp**.

O Firefox inicialmente alertou sobre um processo já em execução e sem resposta. Após fechar o aviso, foi possível utilizar a janela aberta e seguir a configuração sem reiniciar a AttackBox.

### 5.3 Interceptação e histórico

Foi realizada uma requisição de teste para `http://neverssl.com/` e observada no Burp:

```http
GET / HTTP/1.1
Host: neverssl.com
```

O botão **Intercept on** reteve a requisição; **Forward** a encaminhou. Posteriormente, desativar a interceptação evitou que o navegador aguardasse indefinidamente por uma liberação.

**Importante:** `Intercept off` permite a passagem do tráfego sem pausa, mas o proxy ainda pode registrar requisições no histórico, conforme as opções configuradas.

## 6. Laboratório de proxy, escopo e site map

### 6.1 Aplicação Bastion Hosting

O laboratório forneceu um host didático, acessado em HTTP, e uma AttackBox separada. O *Site map* revelou caminhos como:

```text
/
/about
/assets
/contact
/privacy
/products
/products/1
/ticket/
/<ENDPOINT_ALEATORIO_DO_LABORATORIO>
```

Durante a navegação, a página de produtos e o formulário de suporte foram observados. A partir do caminho `/ticket/`, o Burp registrou uma chamada a um endpoint de nome não descritivo.

**Evidência confirmada desse endpoint:**

| Elemento | Observado |
|---|---|
| Método | `GET` |
| Status | `200 OK` |
| Tamanho registrado | 215 bytes |
| `Referer` | Página `/ticket/` |
| `Content-Type` da resposta | `text/plain` |
| Resultado | Resposta inspecionada, e desafio confirmado pelo estudante; flag omitida |

**Diagnóstico:** a presença de um endpoint incomum foi verificada pelo *Site map*, sem depender de tentativas aleatórias de URL. Um painel `Response` inicialmente vazio não provava ausência de conteúdo; após a navegação e nova captura, uma resposta foi observada.

### 6.2 Configuração do escopo

No **Target → Scope**, o endereço-base da aplicação didática foi incluído como prefixo autorizado. Na caixa de diálogo seguinte, foi selecionada a opção para não acumular o histórico de requisições fora do escopo.

Em **Settings → Tools → Proxy → Request interception rules**, foi ativada uma regra:

```text
And | URL | Is in target scope
```

**Prova prática:** com a interceptação geral ligada, uma requisição `GET /` ao host do laboratório ficou pausada.

**Limite da conclusão:** isso demonstra a interceptação do **alvo dentro do escopo**; o histórico não preserva um teste isolado confirmando que todas as requisições fora do escopo passam sem pausa. **Histórico fora de escopo desabilitado** e **interceptação condicionada ao escopo** são controles diferentes.

### 6.3 HTTPS e autoridade certificadora

A sala explicou como o Burp inspeciona tráfego HTTPS quando o cliente confia na autoridade certificadora local do Burp. Foram estudados:

- CA (*Certificate Authority*), certificados e TLS;
- possíveis avisos de confiança do Firefox;
- importação de certificado CA quando necessária;
- riscos de instalar uma CA de interceptação fora de um laboratório controlado.

A AttackBox usada na sala já possuía preparação adequada; **não há evidência de uma instalação manual de certificado nessa prática**.

## 7. Demonstração autorizada de XSS

### 7.1 Objetivo

A **Task 14 — Example Attack** apresentou um formulário de suporte que aplicava restrições do lado do cliente, mas cujo backend não tratava adequadamente uma entrada refletida. O teste foi feito **somente na aplicação de laboratório**.

### 7.2 Requisição inicial

O formulário tinha dois campos: e-mail e mensagem. O envio normal produziu uma requisição `POST /ticket/` com corpo do tipo `application/x-www-form-urlencoded`.

Exemplo com os dados fictícios usados na prática:

```http
POST /ticket/ HTTP/1.1
Host: <HOST_AUTORIZADO_DO_LABORATORIO>
Content-Type: application/x-www-form-urlencoded

email=pentester%40example.com&content=Test+Attack
```

- `%40` representa `@` em URL encoding.
- `&` separa campos codificados em pares chave-valor.
- `+` representa espaço nesse formato.

### 7.3 Modificação via Burp

A requisição foi interceptada no **Proxy → Intercept**. O valor de `email` foi substituído por uma entrada JavaScript demonstrativa, mantendo o campo de mensagem. A requisição modificada foi encaminhada com **Forward**; em seguida, o navegador realizou uma nova requisição `GET /ticket/`, que também precisou ser liberada enquanto a interceptação continuava ligada.

**Evidência observada:** o navegador exibiu uma caixa `alert()` com a mensagem de teste, confirmando que o JavaScript foi executado naquele fluxo.

Por segurança do portfólio, **o payload completo não é reproduzido**: o aprendizado está na interpretação da requisição e do resultado, não em publicar uma resposta de desafio.

### 7.4 Interpretação e mitigação

A demonstração evidenciou que controles de entrada exclusivamente no navegador podem ser contornados quando a requisição é alterada antes de chegar ao servidor.

Proteções importantes incluem:

- Validação de dados no servidor, de acordo com o contrato da aplicação;
- **codificação de saída sensível ao contexto** de HTML/atributos/JavaScript;
- evitar inserir dados não confiáveis diretamente em `innerHTML`;
- CSP como mitigação complementar, não substituto para correção do código;
- uso apropriado de `HttpOnly`, `Secure` e outras proteções de sessão.

O teste demonstrou execução de script na aplicação do laboratório; **não demonstrou** comprometimento de usuários reais, extração de cookies ou persistência de código.

## 8. Dificuldades, correções e decisões técnicas

| Ocorrência | O que aprendemos |
|---|---|
| Firefox alertou sobre instância sem resposta | Evitar reiniciar a AttackBox de imediato; verificar se o navegador aberto pode ser usado |
| Ícone do Burp não estava no desktop | Encontrar o programa por **Applications → Web** |
| Navegação ficou carregando indefinidamente | `Intercept on` estava segurando requisições; liberar ou desligar interceptação conforme objetivo |
| Tráfego de `neverssl.com` e detecção de rede do Firefox apareceu no Burp | Nem todo tráfego capturado pertence ao alvo; definir escopo para manter o foco |
| Resposta de `/ticket/` inicialmente vazia no painel do Burp | Não inferir ausência de conteúdo; verificar status de captura e refazer acesso se necessário |
| Endpoint incomum encontrado no Site map | Investigar o contexto da requisição, inclusive o `Referer`, antes de concluir |
| Confusão sobre o papel do Burp no SOC | Diferenciar investigação defensiva baseada em logs de testes ativos autorizados |
| Questão de configuração do Burp com respostas `yea/nay` | Ler literalmente o formato solicitado, sem assumir `yes/no` |
| Erros SQL 1064 e 1193 na outra conversa | Revisar sintaxe e executar instruções completas (`UPDATE ... SET ... WHERE ...`) |

## 9. Glossário de comandos, métodos e siglas

| Termo | Significado / finalidade |
|---|---|
| HTTP | Hypertext Transfer Protocol; comunicação web |
| HTTPS | HTTP sobre TLS, com proteção em trânsito |
| URL | Uniform Resource Locator; endereço de recurso |
| URI / Path | Identificador/caminho do recurso na aplicação |
| Query string | Parâmetros na URL após `?` |
| API | Application Programming Interface |
| WAF | Web Application Firewall |
| GET | Solicitar representação de recurso |
| POST | Enviar dados para processamento |
| PUT / PATCH | Substituir / modificar parcialmente recurso, conforme API |
| DELETE | Solicitar exclusão de recurso |
| OPTIONS | Consultar opções de comunicação |
| `Host` | Informa a autoridade de destino na requisição HTTP/1.1 |
| `User-Agent` | Identifica características declaradas do cliente |
| `Referer` | Informa a URL de referência quando enviada |
| `Content-Type` | Indica tipo de mídia do corpo |
| `Content-Length` | Tamanho do corpo em bytes, quando aplicável |
| `Set-Cookie` | Define cookie na resposta |
| `Secure` / `HttpOnly` | Restrições ao envio / ao acesso de cookies por JavaScript |
| CSP | Content Security Policy |
| HSTS | HTTP Strict Transport Security |
| `nosniff` | Diretiva de `X-Content-Type-Options` |
| DOM | Document Object Model |
| XSS | Cross-Site Scripting; execução indevida de script no contexto web |
| TLS | Transport Layer Security |
| CA | Certificate Authority |
| Proxy | Intermediário de comunicação |
| FoxyProxy | Extensão que configura o navegador para usar um proxy |
| Burp Intercept | Pausa requisições para análise/edição manual |
| Burp Forward | Encaminha requisição interceptada |
| Burp Drop | Descarta requisição interceptada |
| HTTP history | Registro do tráfego observado pelo Burp |
| Site map | Mapa organizado dos recursos observados |
| Target Scope | Escopo dos sistemas autorizados |
| SQL | Structured Query Language |
| CRUD | Create, Read, Update, Delete |
| DBMS | Database Management System |
| Primary/Foreign Key | Identificador de registro / referência entre tabelas |
| `SELECT` | Consultar dados |
| `INSERT` | Inserir dados |
| `UPDATE ... SET ... WHERE` | Atualizar linhas selecionadas |
| `DELETE ... WHERE` | Excluir linhas selecionadas |
| `console.log()` | Escreve informação no console JavaScript |
| `document.getElementById()` | Seleciona elemento do DOM pelo ID |
| `alert()` / `prompt()` / `confirm()` | Interfaces simples de diálogo em JavaScript |

## 10. Aplicações para SOC Nível 1

1. **Leitura de logs web:** interpretar método, host, path, status, `User-Agent`, IP de origem, horário e parâmetros, considerando limitações da telemetria disponível.
2. **Investigação de anomalias:** sequências de `404`, padrões repetitivos de autenticação, `5xx` inesperados e entradas semelhantes a scripts podem justificar triagem — nenhum elemento isolado confirma ataque.
3. **Proteção de sessão:** avaliar cookies e headers de segurança e comunicar riscos à equipe responsável pela aplicação.
4. **Correlação SIEM/EDR/proxy:** combinar eventos de navegação, DNS, firewall e endpoint para avaliar alertas de acesso suspeito.
5. **Escopo e autorização:** observação passiva e análise de logs normalmente precedem testes ativos. Acessar um site externo suspeito **não dá permissão** para testá-lo com Burp.
6. **Escalonamento técnico:** registrar hipótese, evidência, horário, escopo, impacto provável e recomendações, encaminhando falhas de aplicação à equipe apropriada.

**Posicionamento profissional:** Burp Suite é uma competência complementar para SOC N1, não seu console principal. SIEM, EDR, rede, logs e triagem de incidentes costumam ser mais centrais à função.

## 11. Evidências, limitações e conclusão

### Evidências observadas ou confirmadas pelo estudante

- Exercício HTTP GET retornando `200 OK` e lista de usuários no simulador de API.
- Exercício POST enviado com `country=US` para atualizar o registro didático; questão validada.
- Prática de criação e manipulação de tabela de livros no MySQL, incluindo correção de erros de SQL, recuperada de outra conversa.
- DOM em JavaScript alterado para mostrar o resultado de uma soma.
- FoxyProxy ativado, listener Burp `127.0.0.1:8080`, requisição `GET` interceptada e encaminhada.
- Site map de Bastion Hosting mapeado, endpoint incomum com `200 OK` e resposta analisada.
- Scope configurado, e requisição do laboratório efetivamente interceptada após ativar a regra.
- `POST /ticket/` modificado em laboratório e alerta JavaScript de teste efetivamente exibido no Firefox.
- Conclusão do módulo Web Hacking comunicada pelo estudante.

### Limitações

- A conversa anterior com parte de JavaScript/SQL foi recuperada por registros de contexto, não por uma transcrição integral de todas as telas; por isso algumas seções são apresentadas como conceitos abordados.
- A Task 10 de Web Application Basics não possui, no histórico recuperado, resposta final do `DELETE` para documentação detalhada.
- A conclusão geral do módulo foi **informada pelo estudante**; este documento não verifica independentemente o painel de progresso do TryHackMe.
- O resultado da demonstração XSS comprova a execução de uma janela de alerta no contexto do laboratório, não um incidente real.

### Conclusão

Web Hacking conectou os fundamentos da web à análise prática de requisições HTTP/HTTPS e ao entendimento de comportamentos inseguros. Além de aprender a estrutura de URLs, cabeçalhos, métodos, códigos de status, JavaScript e SQL, a prática com o Burp Suite mostrou a diferença entre **observar, interceptar, modificar e encaminhar** uma requisição.

O principal aprendizado para o objetivo **SOC Analyst Jr. / Blue Team** foi adquirir vocabulário e raciocínio técnico para reconhecer padrões de ataques web em evidências, respeitando sempre **escopo, autorização e limites de atuação**.

---

*Relatório educacional preparado para o portfólio GitHub. Laboratórios autorizados TryHackMe, sem divulgação de flags, credenciais ou segredos.*
