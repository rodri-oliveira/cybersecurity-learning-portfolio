# Networking — Fundamentos, análise de tráfego e reconhecimento de rede

**Trilha:** TryHackMe — Cyber Security 101  
**Área:** Networking / Segurança da Informação / Blue Team  
**Registro de estudos:** outubro–novembro de 2026  
**Objetivo profissional:** desenvolver fundamentos práticos para atuação como Analista SOC Júnior.

> **Escopo e integridade das evidências:** este relatório consolida atividades discutidas e praticadas durante a trilha de Networking, incluindo *Networking Concepts*, *Networking Secure Protocols*, *Wireshark: The Basics*, *Tcpdump: The Basics* e *Nmap: The Basics*. Os exemplos são restritos aos ambientes autorizados do TryHackMe. Resultados confirmados no terminal são indicados como evidências; conceitos apresentados sem registro de execução são tratados como conhecimento estudado. Não são publicadas flags, senhas ou dados de autenticação.

## Sumário

1. [Objetivos e competências](#1-objetivos-e-competências)
2. [Fundamentos de redes](#2-fundamentos-de-redes)
3. [Protocolos seguros](#3-protocolos-seguros)
4. [Análise de pacotes com Wireshark](#4-análise-de-pacotes-com-wireshark)
5. [Análise de pacotes com Tcpdump](#5-análise-de-pacotes-com-tcpdump)
6. [Reconhecimento e varredura com Nmap](#6-reconhecimento-e-varredura-com-nmap)
7. [Erros, correções e lições aprendidas](#7-erros-correções-e-lições-aprendidas)
8. [Aplicações em SOC / Blue Team](#8-aplicações-em-soc--blue-team)
9. [Glossário de comandos](#9-glossário-de-comandos)
10. [Conclusão e próximos passos](#10-conclusão-e-próximos-passos)

---

## 1. Objetivos e competências

Este conjunto de laboratórios teve como propósito:

- Entender endereçamento IPv4, sub-redes, protocolos e portas.
- Distinguir mecanismos de comunicação e suas implicações de segurança.
- Identificar requisições e respostas em capturas de tráfego.
- Aplicar filtros de protocolo, tamanho e flags TCP.
- Descobrir hosts ativos, portas abertas e serviços em ambientes controlados.
- Interpretar ferramentas e seus limites, evitando conclusões além das evidências.

**Ferramentas trabalhadas:** Wireshark, Tcpdump, Nmap, terminal Linux e navegador do AttackBox.

## 2. Fundamentos de redes

### 2.1 Modelos, endereços e transporte

Foram revisados fundamentos necessários para interpretar capturas:

| Conceito | Utilidade prática |
|---|---|
| IPv4 | Identificar origem, destino e faixa de rede |
| Máscara CIDR | Delimitar o escopo da rede e as sub-redes |
| MAC / Ethernet | Identificar interfaces em comunicações de camada 2 |
| ARP | Descobrir o endereço MAC associado a um IPv4 na rede local |
| ICMP | Observar mensagens de controle, diagnóstico e erros |
| TCP | Comunicação orientada a conexão, portas e flags |
| UDP | Comunicação sem estabelecimento de conexão TCP |
| DNS | Consultar registros para resolução de nomes |

### 2.2 Exemplo de subnetting: `/27`

Na questão da sala *Nmap: The Basics*, foi analisado o alvo `192.168.0.1/27`:

- IPv4 contém 32 bits; com prefixo `/27`, restam **5 bits** para endereçamento.
- `2^5 = 32` endereços totais por sub-rede.
- Bloco que contém `192.168.0.1`: `192.168.0.0` a `192.168.0.31`.
- Endereço da rede: `192.168.0.0`.
- Broadcast: `192.168.0.31`.
- Faixa normalmente utilizável por hosts: `192.168.0.1` a `192.168.0.30`.

**Aprendizado:** o último endereço do bloco é diferente do último endereço normalmente atribuível a um host.

## 3. Protocolos seguros

Na sala *Networking Secure Protocols*, estudou-se a finalidade de protocolos que protegem dados em trânsito.

| Tema | Entendimento construído |
|---|---|
| TLS / HTTPS | TLS fornece proteção criptográfica para a comunicação; HTTPS é HTTP sobre TLS |
| Certificados | Participam da autenticação do servidor e da confiança na conexão |
| SSH | Permite acesso remoto com canal criptografado |
| SFTP | Transferência de arquivos sobre SSH, comumente na porta 22 |
| FTPS | FTP protegido por TLS; distinto de SFTP |
| IMAP/POP3 | Protocolos de e-mail que exigem cuidado com versões sem criptografia |
| VPN | Cria um canal protegido entre pontos de comunicação |

### 3.1 Exercício com tráfego TLS no Wireshark

Na análise de uma captura de laboratório, foi utilizado um arquivo de registro de chaves TLS (`ssl-key.log`) configurado no Wireshark para permitir a interpretação de tráfego HTTPS de teste.

**Lição técnica:** uma captura criptografada, sozinha, normalmente não expõe o conteúdo de aplicação. A descriptografia naquele exercício só foi possível porque o laboratório forneceu material de chaves apropriado. Isso não representa capacidade de descriptografar arbitrariamente sessões TLS de terceiros.

**Cuidados:** arquivos de chaves de sessão e conteúdo de autenticação são sensíveis e **não devem ser enviados ao GitHub**.

## 4. Análise de pacotes com Wireshark

Na sala *Wireshark: The Basics*, o foco foi a interface gráfica e a inspeção estruturada de pacotes.

### 4.1 Funcionalidades estudadas

- **Packet List:** relação de pacotes capturados.
- **Packet Details:** campos dos protocolos por camada.
- **Packet Bytes:** conteúdo bruto em representação hexadecimal/ASCII.
- **Display Filters:** filtros que alteram a visualização sem modificar a captura original.
- **Follow HTTP Stream:** acompanhamento de uma conversa HTTP.
- **Export Objects → HTTP:** exportação de objetos identificados em tráfego HTTP.
- **Statistics / Capture File Properties:** metadados e informações gerais do arquivo.

### 4.2 Evidências discutidas no laboratório

- Uma captura de exercícios apresentou **58.620 pacotes**.
- A aplicação do filtro de exibição `http` reduziu o conjunto mostrado a **1.089 pacotes** (aproximadamente 1,9%).
- Foram analisados campos HTTP, IPv4, TCP, conteúdo de aplicação e metadados dos quadros (*frames*).
- Na inspeção de um pacote HTTP, foi identificado **TTL 47** e **payload TCP de 424 bytes**.
- Objetos de uma conversa HTTP foram inspecionados com as funções de acompanhamento de fluxo e exportação.

### 4.3 Método de análise

1. Identificar a pergunta do laboratório e o protocolo pertinente.
2. Aplicar um filtro de exibição adequado.
3. Selecionar um pacote de interesse.
4. Expandir as camadas para distinguir cabeçalhos, valores e payload.
5. Confirmar o significado dos campos antes de registrar uma conclusão.

**Lição:** o tipo declarado no cabeçalho HTTP não substitui a inspeção do conteúdo real do pacote; campos semelhantes podem aparecer em camadas diferentes.

## 5. Análise de pacotes com Tcpdump

Na sala *Tcpdump: The Basics*, foi utilizado o arquivo `traffic.pcap`.

### 5.1 Contagem de pacotes ICMP

```bash
tcpdump -r traffic.pcap icmp 2>/dev/null | wc -l
```

**Resultado registrado:** `26` linhas de pacotes ICMP.

- `-r traffic.pcap`: lê uma captura existente.
- `icmp`: aplica filtro de protocolo.
- `2>/dev/null`: oculta mensagens enviadas ao fluxo de erro padrão.
- `|`: encaminha a saída para o próximo comando.
- `wc -l`: conta linhas.

### 5.2 Requisição ARP e identificação do solicitante

```bash
tcpdump -n -r traffic.pcap arp
```

Trecho observado:

```text
ARP, Request who-has 192.168.124.137 tell 192.168.124.148
ARP, Reply 192.168.124.137 is-at 52:54:00:23:60:2b
```

**Interpretação:** `192.168.124.148` é o host que perguntou; `192.168.124.137` é o IP procurado. A expressão `tell` não identifica o destino da consulta, e sim o solicitante.

### 5.3 Identificação de MAC com cabeçalho Ethernet

```bash
tcpdump -n -r traffic.pcap -e arp
```

Na requisição ARP observada:

```text
52:54:00:7c:d3:5b > ff:ff:ff:ff:ff:ff
Request who-has 192.168.124.137 tell 192.168.124.148
```

- MAC de origem da requisição: `52:54:00:7c:d3:5b`.
- MAC de destino: `ff:ff:ff:ff:ff:ff` (*broadcast* Ethernet).
- `-e` adiciona detalhes de cabeçalho de enlace.

### 5.4 Consultas DNS

```bash
tcpdump -n -r traffic.pcap 'udp port 53'
```

Primeira consulta DNS observada no filtro:

```text
192.168.124.137.33672 > 192.168.124.1.53:
39913+ A? mirrors.rockylinux.org.
```

- `A?`: consulta por registro IPv4.
- `AAAA?`: consulta por registro IPv6.
- Porta de destino `53`: serviço DNS nesse tráfego.
- A direção `origem > destino` diferencia consulta e resposta.

**Limitação:** filtrar `udp port 53` não cobre necessariamente DNS sobre TCP ou outras modalidades como DNS criptografado.

### 5.5 Filtro TCP de flags RST

```bash
tcpdump -n -r traffic.pcap 'tcp[tcpflags] == tcp-rst' 2>/dev/null | wc -l
```

**Resultado registrado:** `57` pacotes com o campo de flags TCP igual somente à flag RST.

O operador `==` é importante: pacotes com combinações como `RST, ACK` não atendem à comparação exata.

### 5.6 Pacotes grandes e análise da origem

```bash
tcpdump -nn -c 5 -r traffic.pcap 'greater 15000'
```

As primeiras cinco linhas retornadas apresentaram origem `185.117.80.53`, porta TCP `80`, destino `192.168.124.137`.

Para deduplicar a coluna de origem:

```bash
tcpdump -nn -r traffic.pcap 'greater 15000' 2>/dev/null | awk '/ IP / {print $3}' | sort -u
```

**Resultado observado:** `185.117.80.53.80`.

**Ressalva relevante:** embora a análise da captura tenha apontado essa origem, a plataforma rejeitou a resposta inserida no exercício. O motivo da divergência **não foi confirmado** nos registros disponíveis; por isso, este resultado é apresentado como **evidência do arquivo analisado**, não como resposta validada pelo TryHackMe.

### 5.7 Exibição de pacotes

Opções revistas na sala:

| Opção | Uso |
|---|---|
| `-q` | Exibição resumida |
| `-e` | Cabeçalho Ethernet (inclui MACs) |
| `-A` | Payload representado em ASCII |
| `-x` | Dados em hexadecimal |
| `-X` | Hexadecimal e ASCII |

## 6. Reconhecimento e varredura com Nmap

Na sala *Nmap: The Basics*, foram vistos descoberta de hosts, varreduras TCP/UDP, versões de serviços, modelos de timing e formatos de saída.

### 6.1 Métodos de varredura

| Opção | Nome | Explicação |
|---|---|---|
| `-sn` | Host discovery | Descobre hosts sem varredura de portas |
| `-sT` | TCP Connect | Tenta estabelecer a conexão TCP completa |
| `-sS` | TCP SYN | Verifica estado de portas com sondagens SYN, normalmente com privilégios adequados |
| `-sU` | UDP scan | Verifica portas UDP |
| `-p` | Seleção de portas | Limita o conjunto de portas investigadas |
| `-O` | OS detection | Tenta identificar o sistema operacional |
| `-sV` | Service version | Busca identificar versões de serviços |
| `-Pn` | Sem host discovery | Trata o alvo como ativo e prossegue à varredura |
| `-T4` | Timing aggressive | Ajusta a estratégia de temporização |
| `-d` | Debug | Aumenta as informações de depuração |

### 6.2 Caso prático: distinção entre AttackBox e máquina-alvo

Ocorreu um erro importante de escopo: foi inicialmente executado `nmap -sT` contra o próprio IP do AttackBox, e não contra o IP do alvo. O resultado inicial não representava a máquina solicitada pelo laboratório.

Após confirmar a seção de máquinas ativas do TryHackMe, foram distinguidos:

| Papel | IP observado naquela sessão |
|---|---|
| AttackBox | `10.64.76.254` |
| Máquina-alvo | `10.64.154.159` |

Esses endereços são **temporários do laboratório** e podem mudar entre sessões.

### 6.3 Varredura TCP Connect no alvo correto

Comando executado:

```bash
nmap -sT 10.64.154.159
```

Trecho do resultado observado:

```text
Host is up (0.00050s latency).
Not shown: 994 closed tcp ports (conn-refused)
PORT     STATE SERVICE
7/tcp    open  echo
9/tcp    open  discard
13/tcp   open  daytime
17/tcp   open  qotd
22/tcp   open  ssh
8008/tcp open  http
```

**Evidência:** seis portas TCP abertas entre as 1.000 portas padrão verificadas. A coluna `SERVICE` indica associações conhecidas para números de porta; não garante, sozinha, a identidade real do software.

### 6.4 Serviço web em porta não padrão

A descoberta de `8008/tcp open http` motivou a orientação de acessar, dentro do laboratório autorizado:

```text
http://10.64.154.159:8008/
```

**Aprendizado:** aplicações web não necessariamente usam apenas as portas 80 e 443. Não há evidência registrada aqui da contagem de elementos visuais da página; portanto, essa resposta não é incluída.

### 6.5 Identificação de versão

Exemplo de procedimento indicado na sala:

```bash
nmap -sV -p 8008 10.64.154.159
```

- `-sV`: tenta identificar produto e versão.
- `-p 8008`: restringe a verificação à porta indicada.

**Status:** a sintaxe foi estudada e recomendada; a saída da execução não foi disponibilizada no histórico analisado. Não se atribui nome ou versão do servidor sem evidência.

### 6.6 Timing e apresentação dos resultados

O Nmap aceita templates `-T0` a `-T5`, de mais conservador a mais agressivo. O modelo `-T4` também pode ser escrito como `-TAggressive`.

Outras opções apresentadas:

```bash
nmap -v <IP_AUTORIZADO>          # maior detalhamento
nmap -d <IP_AUTORIZADO>          # depuração
nmap -oN resultado.txt <IP_AUTORIZADO>  # relatório normal
nmap -oX resultado.xml <IP_AUTORIZADO>  # relatório XML
nmap -oG resultado.gnmap <IP_AUTORIZADO>  # formato grepable
nmap -oA resultado <IP_AUTORIZADO>  # gera vários formatos
```

**Nota:** as linhas acima ilustram opções apresentadas na sala; não representam execuções comprovadas.

## 7. Erros, correções e lições aprendidas

### 7.1 Ordem dos argumentos do Tcpdump

Foi tentado:

```bash
tcpdump -r -n traffic.pcap arp
```

Apareceu erro indicando que `-n` não existia como arquivo. O motivo é que `-r` exige imediatamente o nome da captura.

Correção:

```bash
tcpdump -n -r traffic.pcap arp
```

**Lição:** opções com argumentos obrigatórios devem ser interpretadas corretamente; a ordem é relevante quando há associação posicional entre opção e argumento.

### 7.2 Grandes volumes de saída

Capturas extensas tornaram a leitura manual pouco eficiente. Foram usados:

- `-c 5` para exibir uma amostra pequena de pacotes.
- `wc -l` para contar linhas/pacotes após a filtragem.
- `awk` e `sort -u` para extrair e agrupar origens.

**Lição:** distinguir análise amostral de conferência do conjunto completo. Cinco linhas não provam que toda a captura possui apenas uma origem.

### 7.3 Falha de validação em resposta do exercício

O laboratório rejeitou um endereço de origem que era consistente com as linhas de tráfego observadas. Como não há confirmação da causa da rejeição, o relatório conserva tanto a evidência quanto a incerteza.

**Lição:** nunca afirmar que uma resposta foi validada quando a interface mostrou o contrário.

### 7.4 Alvo incorreto no Nmap

A primeira varredura TCP Connect foi executada contra o IP do AttackBox. Após a conferência visual dos dois endereços do ambiente, a execução foi refeita no alvo correto.

**Lição de segurança:** confirmar escopo, autorização e identidade do destino **antes de cada varredura**. A legitimidade da ferramenta não elimina o risco de analisar o equipamento errado.

## 8. Aplicações em SOC / Blue Team

As técnicas praticadas podem contribuir para investigações defensivas:

| Situação | Técnica aplicável | Cuidado analítico |
|---|---|---|
| Identificar quem iniciou uma comunicação local | ARP e cabeçalho Ethernet | Distinguir solicitante de respondente |
| Verificar consultas de domínio | DNS com Tcpdump ou Wireshark | Não confundir consulta, resposta e outros tipos de DNS |
| Investigar resets de conexão | Flags TCP RST | RST não é prova isolada de ataque |
| Localizar transferências incomuns | Filtros por tamanho | Volume ou tamanho elevados não equivalem a exfiltração |
| Confirmar superfície de rede | Descoberta de portas com Nmap | Uma porta aberta não prova vulnerabilidade |
| Levantar versões para triagem | `nmap -sV` em ativos autorizados | Validar a identificação antes de relacionar CVEs |

Um fluxo inicial de investigação pode ser: **definir escopo → obter evidência → filtrar → interpretar → validar → documentar**.

## 9. Glossário de comandos

| Comando / símbolo | Significado |
|---|---|
| `tcpdump` | Captura ou lê pacotes de rede |
| `-r arquivo.pcap` | Ler pacotes já gravados |
| `-n` / `-nn` | Evitar tradução de nomes; `-nn` também evita tradução de números de portas |
| `-e` (Tcpdump) | Exibir informações de enlace |
| `-c N` (Tcpdump) | Parar após N pacotes correspondentes |
| `greater N` | Selecionar pacotes com comprimento capturado ≥ N |
| `tcp[tcpflags]` | Acessar o campo de flags TCP no filtro |
| `==` | Comparação de igualdade |
| `2>/dev/null` | Redirecionar mensagens de erro para descarte |
| `\|` | Encadear saída padrão de um comando à entrada de outro |
| `wc -l` | Contar linhas |
| `awk '{print $3}'` | Extrair a terceira coluna de cada linha |
| `sort -u` | Ordenar e retirar duplicatas |
| `nmap` | Mapear hosts, portas e serviços |
| `-sT` (Nmap) | TCP Connect Scan |
| `-sV` (Nmap) | Detecção de versões de serviços |
| `-p` (Nmap) | Definir porta ou faixa de portas |
| `-T4` (Nmap) | Template de timing agressivo |
| `-d` (Nmap) | Depuração |

## 10. Conclusão e próximos passos

A etapa de Networking forneceu uma base para entender **como os dispositivos se comunicam**, **como observar esse tráfego** e **como investigar serviços expostos em ambientes autorizados**.

### Conhecimentos demonstrados por evidências

- Filtragem de pacotes ICMP, ARP, DNS e TCP em arquivo PCAP.
- Interpretação de IP de origem, IP de destino, MAC de origem e broadcast.
- Contagem de pacotes e redução de saídas extensas no terminal Linux.
- Uso de filtros por tamanho e flags TCP.
- Execução de varredura TCP Connect e interpretação de portas abertas.
- Correção de erros de sintaxe e de seleção de alvo.

### Próximas competências para o portfólio SOC

- Identificar padrões suspeitos em logs de rede e capturas PCAP.
- Correlacionar IPs, portas, protocolos e timestamps em análises de eventos.
- Estudar SIEM, EDR e automação de resposta em laboratório isolado.
- Elaborar relatórios de incidentes com evidências, hipóteses, limitações e conclusões verificáveis.

---

**Critérios de publicação:** não incluir flags do TryHackMe, senhas, tokens, chaves TLS, conteúdo de autenticação nem dados de terceiros. Os IPs mostrados correspondem a ambientes temporários de laboratório e servem exclusivamente à documentação educacional.
