# Snort Challenge — Live Attacks | TryHackMe

> **Trilha:** Cyber Security / Blue Team · **Laboratório:** Snort Challenge — Live Attacks  
> **Ferramentas:** Snort 2.9.7.0, Linux, `tcpdump`, `ss`, `ip`, `nano`  
> **Escopo:** ambiente controlado e autorizado do TryHackMe

## 1. Objetivo

Investigar tráfego suspeito em dois cenários simulados, identificar a comunicação relacionada ao ataque, criar regras personalizadas no Snort e executar a ferramenta no modo **IPS (Intrusion Prevention System — Sistema de Prevenção de Intrusão)**.

Os cenários trabalhados foram:

1. **Task 2 — Ataque de força bruta:** múltiplas conexões ao serviço SSH.
2. **Task 3 — Tráfego de saída suspeito:** comunicação associada, no contexto da sala, a uma possível *reverse shell*.

**Resultado observado:** em ambos os cenários, após a configuração do Snort em modo inline, o laboratório gerou um arquivo `flag.txt` na área de trabalho. Os valores das flags não são incluídos neste relatório.

## 2. Conceitos fundamentais

| Conceito | Definição | Aplicação no laboratório |
|---|---|---|
| **IDS** (*Intrusion Detection System*) | Sistema de Detecção de Intrusão: observa tráfego e produz alertas. | Análise inicial dos pacotes. |
| **IPS** (*Intrusion Prevention System*) | Sistema de Prevenção de Intrusão: pode impedir tráfego correspondente às regras. | Uso de regras com ação `drop`. |
| **SSH** (*Secure Shell*) | Protocolo para acesso remoto seguro, comumente na porta TCP 22. | Serviço investigado na Task 2. |
| **TCP** (*Transmission Control Protocol*) | Protocolo de transporte orientado à conexão. | Protocolo dos dois fluxos investigados. |
| **SYN** (*Synchronize*) | Sinalizador TCP usado no estabelecimento de conexões. | Identificação de várias tentativas de conexão SSH. |
| **Reverse shell** | Sessão remota iniciada da máquina comprometida para outro host. | Hipótese contextual da Task 3. |
| **SID** (*Signature ID*) | Identificador numérico de uma regra do Snort. | `sid:1000001` e `sid:1000002` nos respectivos ambientes. |
| **DAQ** (*Data Acquisition*) | Mecanismo utilizado pelo Snort para capturar/processar pacotes. | `afpacket` em modo inline. |

## 3. Ambiente e metodologia

- **Sistema:** Ubuntu disponibilizado pelo TryHackMe.
- **Snort:** versão `2.9.7.0 GRE` (identificada na saída do programa).
- **Configuração principal:** `/etc/snort/snort.conf`.
- **Regras personalizadas:** `/etc/snort/rules/local.rules`.
- **Interfaces do laboratório:** `eth0` e `eth1`.
- **Método:** descobrir interfaces → observar conexões → capturar tráfego → identificar origem, destino e porta → criar regra → executar Snort inline → conferir evidências.

> Os endereços IP apresentados são **temporários do laboratório**, não ativos de uma empresa real. A simples presença de tráfego em uma porta não comprova, isoladamente, atividade maliciosa.

## 4. Task 2 — Investigação de força bruta contra SSH

### 4.1 Reconhecimento da máquina e tráfego

A máquina usada nessa etapa apresentava `eth0` com endereço `10.64.155.223` e `eth1` com endereço `10.234.0.1`.

Comandos executados durante a investigação:

```bash
ip -br addr
sudo ss -tn
sudo snort -v -i eth0 'tcp' -n 10
sudo snort -dev -i eth0 'tcp port 80' -n 10
sudo ss -lntp
ps -fp 831,1534
```

**Aprendizado de triagem:** inicialmente apareceu tráfego na porta 80, mas a análise do processo associado mostrou `python3 -m websockify`, componente de encaminhamento WebSocket ligado ao ambiente. Esse tráfego não deveria ser classificado automaticamente como ataque.

Em seguida, observamos outras comunicações:

```bash
sudo timeout 10 tcpdump -n -i eth1 -c 10
sudo timeout 10 tcpdump -nn -i eth0 -c 20 'tcp and not port 80'
sudo timeout 15 tcpdump -nn -i eth0 'tcp dst port 22 and tcp[tcpflags] & tcp-syn != 0'
sudo snort -v -i eth0 -n 10 'tcp dst port 22'
```

A amostra de `eth1` não mostrou pacotes no intervalo observado; o tráfego relevante apareceu no monitoramento de `eth0`.

### 4.2 Evidência do fluxo suspeito

Trecho representativo da captura:

```text
10.10.245.36.46680 > 10.10.140.29.22: Flags [S]
10.10.245.36.46682 > 10.10.140.29.22: Flags [S]
10.10.245.36.46684 > 10.10.140.29.22: Flags [S]
```

| Campo | Valor |
|---|---|
| Origem observada | `10.10.245.36` |
| Destino observado | `10.10.140.29` |
| Porta de destino | `22` |
| Protocolo | `TCP` |
| Serviço | `SSH` |

A captura filtrada registrou **27 pacotes de início de conexão (SYN)** em aproximadamente 15 segundos de monitoramento, com sucessivas portas de origem. No contexto do exercício, esse padrão é compatível com atividade automatizada contra SSH.

**Limite da evidência:** pacotes SYN não demonstram, por si só, tentativas de senha ou autenticações malsucedidas; o conteúdo de autenticação SSH é criptografado.

### 4.3 Arquivo de regras e configuração

As verificações executadas foram:

```bash
ls -l /etc/snort
ls -l /etc/snort/rules
cat /etc/snort/rules/local.rules
grep -n 'local.rules' /etc/snort/snort.conf
grep -n '^var RULE_PATH' /etc/snort/snort.conf
```

Constatou-se que a configuração referencia o arquivo de regras locais:

```text
var RULE_PATH /etc/snort/rules
include $RULE_PATH/local.rules
```

A regra criada na **Task 2**, no arquivo `/etc/snort/rules/local.rules`, foi:

```snort
drop tcp 10.10.245.36 any -> 10.10.140.29 22 (msg:"Block SSH Attack"; sid:1000001; rev:1;)
```

**Interpretação:** `drop` determina descarte quando a ferramenta opera em modo IPS; `tcp` seleciona o protocolo; `any` permite qualquer porta de origem; `->` indica direção; `msg` define a mensagem; `sid` identifica a regra; `rev` marca sua revisão.

### 4.4 Teste, dificuldade e correção

Foi testada a configuração em modo passivo:

```bash
sudo snort -T -c /etc/snort/snort.conf -i eth0
```

A saída indicou:

```text
0 Snort rules read
Snort successfully validated the configuration!
```

Embora a sintaxe geral fosse validada, a regra `drop` não aparecia no conjunto carregado nessa execução passiva. O próprio arquivo local foi conferido com `cat -n` e continha a regra.

Houve também uma tentativa incorreta com a opção `-R` para passar o caminho do arquivo de regras. Na versão utilizada, essa opção foi interpretada como **sufixo do arquivo PID**, resultando em `Invalid pidfile suffix`. A opção não foi utilizada na solução final.

Outro contratempo foi um editor `nano` que permaneceu aberto com um bloqueio de edição. O processo foi identificado por `ps -fp 2366`; a edição foi retomada e a regra efetivamente salva.

### 4.5 Execução em modo IPS

Comando utilizado:

```bash
sudo snort -Q --daq afpacket -i eth0:eth1 -c /etc/snort/snort.conf -A full -l /var/log/snort
```

Mensagens relevantes:

```text
1 Snort rules read
1 detection rules
afpacket DAQ configured to inline.
Acquiring network traffic from "eth0:eth1".
Commencing packet processing
```

O Snort processou **24.533 pacotes** em aproximadamente **5 minutos e 30 segundos**; registrou **114 alertas** e **5.778 vereditos `Blacklist`**. O relatório da execução também mostrou `Block: 0` no contador de vereditos. Esses contadores devem ser interpretados separadamente: não autorizam afirmar que exatamente 5.778 pacotes foram bloqueados pela regra personalizada.

**Evidência de conclusão do cenário:** após a execução, foi identificado um `flag.txt` na pasta `~/Desktop` (37 bytes). O conteúdo não foi publicado.

## 5. Task 3 — Investigação de tráfego de saída / possível reverse shell

### 5.1 Identificação da máquina correta

O primeiro comando `ip -br addr` foi executado inadvertidamente na AttackBox (`10.64.111.113`). A análise foi corrigida, passando para a máquina da **Task 3**, cujo `eth0` possuía o endereço `10.64.129.137`.

Comandos executados:

```bash
ip -br addr
sudo ss -tnp
sudo timeout 15 tcpdump -nn -i eth0 -c 30 'tcp and not port 80'
```

### 5.2 Evidência do fluxo suspeito

Trecho representativo:

```text
10.10.196.55.54156 > 10.10.144.156.4444: Flags [P.], length 176
10.10.144.156.4444 > 10.10.196.55.54156: Flags [.], length 0
```

| Campo | Valor |
|---|---|
| Origem observada | `10.10.196.55` |
| Porta de origem | `54156` (na amostra) |
| Destino observado | `10.10.144.156` |
| Porta de destino | `4444` |
| Protocolo | `TCP` |

A sala descreveu o cenário como atividade de saída potencialmente relacionada a uma **reverse shell**. A porta TCP 4444 é frequentemente associada ao **Metasploit**; entretanto, a porta e a captura, isoladamente, **não comprovam** qual ferramenta gerou o tráfego.

### 5.3 Regra criada

A regra adicionada na Task 3 foi:

```snort
drop tcp 10.10.196.55 any -> 10.10.144.156 4444 (msg:"Block Reverse Shell"; sid:1000002; rev:1;)
```

O conteúdo foi conferido com:

```bash
cat -n /etc/snort/rules/local.rules
```

A ação `drop` restringiu a regra ao fluxo identificado: origem, destino, protocolo e porta. Nenhuma regra genérica de bloqueio de todo o tráfego foi criada.

### 5.4 Execução e observações

Foi executado novamente:

```bash
sudo snort -Q --daq afpacket -i eth0:eth1 -c /etc/snort/snort.conf -A full -l /var/log/snort
```

O Snort exibiu:

```text
1 Snort rules read
1 detection rules
afpacket DAQ configured to inline.
```

Nesta execução, processou **866 pacotes** em aproximadamente **7 segundos**, registrou **10 alertas** e **157 vereditos `Blacklist`**. O contador `Block` também apareceu como `0`; portanto, não é adequado equiparar esse contador às decisões de blacklist.

**Evidência de conclusão do cenário:** foi gerado um novo arquivo `~/Desktop/flag.txt` (37 bytes) na máquina da Task 3. A flag não é revelada neste documento.

## 6. Referência rápida dos comandos

| Comando ou opção | Finalidade |
|---|---|
| `ip -br addr` | Identificar interfaces e endereços IP de forma resumida. |
| `ss -tnp` | Listar conexões TCP e processos associados. |
| `ss -lntp` | Listar sockets TCP em escuta e seus processos. |
| `tcpdump -nn -i eth0` | Capturar tráfego da interface sem resolução de nomes. |
| `tcpdump -c 30` | Limitar a quantidade de pacotes capturados. |
| `timeout 15` | Encerrar o comando automaticamente após 15 segundos. |
| `snort -v -i eth0 -n 10` | Exibir uma pequena amostra de pacotes pelo Snort. |
| `snort -T -c ...` | Testar a configuração sem iniciar monitoramento contínuo. |
| `nano /etc/snort/rules/local.rules` | Editar as regras personalizadas. |
| `cat -n ...` | Conferir o arquivo com numeração de linhas. |
| `snort -Q` | Habilitar a operação inline para prevenção. |
| `--daq afpacket` | Usar o mecanismo AF_PACKET de aquisição/encaminhamento de pacotes. |
| `-i eth0:eth1` | Definir o par de interfaces da operação inline neste laboratório. |
| `-A full` | Configurar a saída dos alertas em formato completo. |
| `-l /var/log/snort` | Definir o diretório dos registros. |

## 7. Lições aprendidas e cuidados de análise

1. **Separar hipótese de evidência:** portas e conexões aparentemente suspeitas devem ser correlacionadas com o contexto; não basta ver a porta 80, 22 ou 4444 para concluir que houve um ataque.
2. **Diferenciar tráfego do laboratório e tráfego investigado:** o processo `websockify` explicou parte das conexões na porta 80, evitando um bloqueio indevido.
3. **Entender o modo de execução:** a regra `drop` exige operação apropriada de prevenção; o Snort em captura passiva não equivale a um IPS.
4. **Validar o carregamento da regra:** conferir `1 Snort rules read` e `afpacket DAQ configured to inline` antes de avaliar o comportamento.
5. **Interpretar estatísticas com rigor:** `Alerts`, `Dropped`, `Block` e `Blacklist` são contadores diferentes; não devem ser tratados como sinônimos.
6. **Confirmar a máquina correta:** AttackBox e máquina da tarefa possuem IPs diferentes.
7. **Preservar os limites da prova:** a geração do arquivo `flag.txt` comprova a condição de sucesso reconhecida pelo laboratório, mas não substitui uma auditoria independente de cada pacote descartado.

## 8. Relevância para SOC L1

A prática desenvolveu habilidades aplicáveis ao trabalho de um **SOC L1 (Security Operations Center, Level 1 — Centro de Operações de Segurança, Nível 1)**:

- Identificação de origem, destino, serviço, protocolo e porta.
- Análise de comportamento de conexão TCP, incluindo sinalizadores SYN e ACK.
- Diferenciação entre tráfego normal, suspeito e comprovadamente malicioso.
- Construção de regras de detecção/prevenção com escopo delimitado.
- Investigação de erros de configuração e interpretação de logs operacionais.
- Registro técnico com evidências e limitações claramente documentadas.

---

**Nota de publicação:** relatório educacional baseado nos comandos e saídas efetivamente apresentados durante o laboratório. Flags, credenciais e senhas foram omitidas intencionalmente. A aceitação de cada resposta textual pelo botão *Check* do TryHackMe não foi documentada integralmente no histórico; o resultado prático observado foi a geração das flags nos dois cenários.
