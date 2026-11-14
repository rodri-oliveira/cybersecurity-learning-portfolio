# Defensive Security — TryHackMe

> Relatório técnico de estudos e laboratórios | Cyber Security 101 | Blue Team / SOC Level 1

## 1. Visão geral

Este relatório consolida o módulo **Defensive Security** da trilha Cyber Security 101 do TryHackMe, concluído em 14 de novembro de 2026. O objetivo foi desenvolver fundamentos de operações de segurança, investigação forense, resposta a incidentes e análise de logs, com foco na atuação de um analista SOC L1.

### Conteúdo estudado

| Sala | Competências desenvolvidas |
|---|---|
| Introduction to SOC | Triagem, 5 Ws, interpretação de alertas e análise de tráfego em SIEM |
| Digital Forensics Fundamentals | Preservação de evidências, metadados de PDF e EXIF, geolocalização |
| Incident Response Fundamentals | Classificação de incidentes, PICERL, playbooks e contenção em EDR simulado |
| Logs Fundamentals | Windows Event Viewer, correlação de eventos e filtragem de logs HTTP |

**Ambiente:** TryHackMe AttackBox (Linux), máquina virtual Windows e interfaces simuladas de SIEM e EDR. Todos os exercícios foram conduzidos em ambiente educacional autorizado.

## Sumário

- [2. Introduction to SOC](#2-introduction-to-soc)
- [3. Digital Forensics Fundamentals](#3-digital-forensics-fundamentals)
- [4. Incident Response Fundamentals](#4-incident-response-fundamentals)
- [5. Logs Fundamentals](#5-logs-fundamentals)
- [6. Competências e próximos passos](#6-competências-e-próximos-passos)

## 2. Introduction to SOC

### Fundamentos

SOC (Security Operations Center) é a estrutura responsável por monitorar eventos de segurança, detectar ameaças e apoiar a resposta a incidentes. Seus três pilares são **People, Process e Technology**.

| Função | Responsabilidade |
|---|---|
| SOC Analyst L1 | Triagem inicial, coleta de evidências, classificação e reporte de alertas |
| SOC Analyst L2 | Investigação aprofundada e correlação de informações |
| SOC Analyst L3 | Incidentes complexos, causa raiz e apoio à resposta |
| Security Engineer | Implantação e manutenção de soluções de segurança |
| Detection Engineer | Desenvolvimento e aprimoramento de regras de detecção |
| SOC Manager | Gestão da equipe, processos e comunicação |

A triagem utiliza os **5 Ws**: What (o quê), When (quando), Where (onde), Who (quem) e Why (por quê). O reporte organiza as evidências em um ticket e permite encaminhar casos relevantes para investigação ou resposta.

### Tecnologias

- **SIEM (Security Information and Event Management):** centraliza logs, correlaciona eventos e gera alertas.
- **EDR (Endpoint Detection and Response):** oferece visibilidade sobre dispositivos, investigação e ações de resposta.
- **Firewall:** controla o tráfego de rede conforme regras de segurança.

### Laboratório: triagem de varredura de portas

Foi investigado o alerta **#167 — Port Scanning Activity Detected from IP: 10.0.0.8**, classificado como P2 / High Severity. O alerta foi assumido com **Acknowledge** e aberto na opção **Investigate in SIEM**.

A interface apresentou 4.300 registros associados ao alerta. A análise dos registros de tráfego permitiu identificar o scanner, o destino e o horário da atividade.

| Campo | Evidência |
|---|---|
| What | Port Scanning |
| When | 12/06/2024, 17:24 (horário exibido nos logs) |
| Where | 10.0.0.3 — JOE PC |
| Who | 10.0.0.8 — NESSUS |
| Why | Intended — varredura autorizada pela equipe de avaliação de vulnerabilidades |
| Resposta ao scanner | Sim: tráfego de JOE PC para NESSUS, origem 22 e destino 56927 |

**Evidência representativa:**

```text
Origem       Destino      Porta origem  Porta destino
10.0.0.8     10.0.0.3     56927         443
10.0.0.8     10.0.0.3     56927         53
10.0.0.8     10.0.0.3     56927         22
10.0.0.3     10.0.0.8     22            56927
```

A repetição do IP de origem com diferentes portas de destino foi consistente com uma varredura. O registro com origem e destino invertidos demonstrou tráfego de retorno. Esse registro, isoladamente, não comprova uma autenticação SSH bem-sucedida.

**Conclusão:** a detecção identificou corretamente uma varredura, mas o contexto fornecido pela equipe de vulnerabilidades demonstrou que a atividade era autorizada. A severidade inicial do alerta não substitui a avaliação contextual. Trata-se de uma atividade legítima detectada pela regra, não de evidência de invasão.

## 3. Digital Forensics Fundamentals

### Metodologia e preservação de evidências

O modelo apresentado na sala divide a investigação forense em quatro fases:

| Fase | Finalidade |
|---|---|
| Collection | Identificar e coletar evidências preservando os dados originais |
| Examination | Examinar e extrair informações relevantes |
| Analysis | Correlacionar evidências e formular conclusões |
| Reporting | Documentar metodologia, resultados e limitações |

A aquisição deve respeitar autorização adequada, cadeia de custódia e integridade da evidência. Um **write blocker** impede gravações no dispositivo original durante a aquisição. A cadeia de custódia registra responsáveis, datas, armazenamento e acessos à evidência.

A sala abordou imagens de disco (dados não voláteis) e imagens de memória (dados voláteis), além de FTK Imager, Autopsy, DumpIt e Volatility. Essas ferramentas foram estudadas conceitualmente; o exercício executado concentrou-se em metadados.

### Laboratório: análise de carta de resgate e fotografia

Cenário fictício de sequestro de um gato, com uma carta de resgate em PDF e uma fotografia extraída do documento original.

#### Preparação do ambiente

```bash
cd /root/Rooms/introdigitalforensics
ls -l
```

- `cd`: altera o diretório atual.
- `ls -l`: lista arquivos com detalhes de permissões, tamanho e propriedade.

Foram identificados os arquivos `ransom-letter.pdf`, `ransom-letter.doc` e `letter-image.jpg`, entre outros materiais do laboratório.

#### Metadados do PDF

```bash
pdfinfo ransom-letter.pdf
```

`pdfinfo` extrai informações do documento PDF; o argumento indica o arquivo examinado.

| Campo | Resultado observado |
|---|---|
| Title | Pay NOW |
| Subject | We Have Gato |
| Author | Ann Gree Shepherd |
| Creator / Producer | Microsoft Word 2016 |
| CreationDate / ModDate | 23/02/2022, 09:10:36 UTC |
| Pages | 1 |
| PDF version | 1.7 |

O campo Author revelou o nome registrado como autor. Metadados são pistas que podem ser alteradas; não constituem, isoladamente, prova de autoria ou identidade.

#### Metadados EXIF da fotografia

```bash
exiftool letter-image.jpg
```

`exiftool` exibe metadados de arquivos. EXIF significa Exchangeable Image File Format.

| Campo | Resultado observado |
|---|---|
| Make | Canon |
| Camera Model Name | Canon EOS R6 |
| Software | GIMP 2.10.28 |
| GPS Latitude | 51° 30' 51.90" N |
| GPS Longitude | 0° 5' 38.73" W |
| Image Size | 1200 × 800 |

As coordenadas foram consultadas no Google Maps, permitindo identificar **Milk Street, Londres, Reino Unido**, nas proximidades de Cheapside.

**Conclusão:** a análise demonstrou como metadados de documentos e imagens podem revelar informações sobre autoria declarada, equipamento e localização. A presença de histórico de edição e datas divergentes reforça a necessidade de validar a origem e a integridade dos metadados antes de formular conclusões definitivas.

## 4. Incident Response Fundamentals

### Conceitos estudados

Um evento representa uma atividade registrada; um alerta sinaliza uma atividade potencialmente suspeita; um incidente envolve uma ameaça ou violação que exige tratamento. A classificação considera contexto, evidências e impacto.

- **True Positive:** detecção de uma ameaça real.
- **False Positive:** indicação de ameaça que não se confirma.
- **Severidade:** Low, Medium, High e Critical, conforme impacto e contexto.

Foram estudadas infecções por malware, violações de segurança, vazamentos de dados, ataques internos e negação de serviço (DoS).

### Processo de resposta

| SANS — PICERL | Modelo NIST tradicional apresentado na sala |
|---|---|
| Preparation | Preparation |
| Identification | Detection and Analysis |
| Containment | Containment, Eradication and Recovery |
| Eradication | Containment, Eradication and Recovery |
| Recovery | Containment, Eradication and Recovery |
| Lessons Learned | Post Incident Activity |

O modelo de quatro fases corresponde à abordagem tradicional ensinada no laboratório, não à nomenclatura de todas as revisões atuais do NIST.

O **Incident Response Plan** estabelece responsabilidades, comunicação e escalonamento. **Playbooks** orientam o tratamento de categorias de incidentes; **runbooks** detalham a execução de tarefas específicas.

### Laboratório: resposta a incidente de phishing

O cenário simulou um e-mail com anexo malicioso. A análise inicial revelou:

| Campo | Evidência |
|---|---|
| Remetente exibido | Jeff Johnson |
| Assunto | Pay Slip for novembro |
| Anexo | Payslip.pdf |
| Vetor de ameaça | Email Attachment |
| Engenharia social | Pretexto de pagamento salarial |

A divergência entre o mês do assunto (novembro) e o mês mencionado no corpo (abril) foi identificada como sinal de alerta. O endereço exibido e o nome do remetente não comprovam a identidade real de quem enviou a mensagem.

No ambiente simulado, o download do anexo disparou a mensagem **Malware detected on the host!**. A opção **Take Actions** abriu a investigação dos dispositivos que possuíam o arquivo.

| Dispositivo | Estado observado | Tratamento indicado |
|---|---|---|
| Host-ATYU | Not Executed | Quarentena do arquivo |
| Host-IOPE | Not Executed | Quarentena do arquivo |
| Host-HKNV | Executed | Investigação do dispositivo |

**Escopo identificado:** três dispositivos possuíam o anexo, dos quais um havia executado o arquivo.

A prática diferenciou presença de um arquivo de sua execução e apresentou a quarentena como medida para impedir novas execuções. A investigação do dispositivo que executou o arquivo é necessária para determinar impacto, persistência e possíveis atividades adicionais.

**Limite das evidências registradas:** os prints documentam a identificação do malware, o escopo e as ações disponíveis. Não documentam a linha do tempo detalhada do host executado nem a confirmação de todas as ações finais de contenção, erradicação e recuperação. Essas etapas não são apresentadas aqui como verificadas.

## 5. Logs Fundamentals

### Tipos de logs

| Tipo | Utilidade |
|---|---|
| System Logs | Inicialização, serviços, drivers e erros do sistema |
| Security Logs | Autenticação, autorização e alterações de contas |
| Application Logs | Atividades e erros de aplicações |
| Audit Logs | Rastreabilidade de ações e alterações |
| Network Logs | Conexões e tráfego de entrada e saída |
| Access Logs | Acessos a recursos, aplicações e servidores web |

### Laboratório 1 — Windows Event Logs Analysis

Cenário: investigação de um sistema Windows comprometido, relacionado a um incidente de exfiltração de dados de um servidor de arquivos.

Foi utilizado o **Event Viewer**, acessando `Windows Logs > Security > Filter Current Log` para filtrar os registros por Event ID.

| Event ID | Descrição |
|---|---|
| 4720 | Criação de conta de usuário |
| 4722 | Habilitação de conta de usuário |
| 4724 | Tentativa de redefinição de senha |

#### Evidências e linha do tempo

| Data e hora (local do log) | Event ID | Conta / resultado |
|---|---|---|
| 07/06/2024 12:56:27 | 4720 | Administrator criou a conta `hacked` |
| 07/06/2024 12:56:27 | 4722 | Conta `hacked` habilitada — Audit Success |
| 07/06/2024 12:58:40 | 4724 | Administrator redefiniu a senha de `hacked` — Audit Success |

**Correlação:** o SID da conta `hacked` terminou em `-1010` nos eventos examinados. A sessão `Logon ID: 0x7333A` apareceu nos registros de criação e redefinição, associada a Administrator.

- **Subject:** conta que realizou a ação.
- **New Account / Target Account:** conta criada ou afetada.
- **Audit Success:** resultado bem-sucedido da operação auditada.

O filtro 4720 retornou três eventos. O mais recente foi examinado para identificar a conta criada e seu responsável. O filtro 4722 retornou três eventos, e a habilitação de `hacked` foi confirmada nos detalhes. O filtro 4724 retornou dois eventos, e o mais recente confirmou a redefinição bem-sucedida da senha.

**Conclusão:** foi possível reconstruir atividades relacionadas ao gerenciamento de contas. O conjunto de registros é consistente com a criação e preparação de uma nova conta no cenário investigado, mas não comprova, isoladamente, quem controlava a sessão Administrator ou que a conta tenha realizado a exfiltração.

### Laboratório 2 — Web Server Access Logs Analysis

Foi analisado o arquivo `access.log` na AttackBox.

```bash
cd /root/Rooms/logs
```

#### Identificar a última requisição GET para /contact

```bash
grep "GET /contact" access.log
```

- `grep`: busca um padrão em um arquivo.
- `"GET /contact"`: padrão contendo método HTTP e caminho solicitado.
- `access.log`: arquivo de registros analisado.

Foram retornados nove registros. Comparando os timestamps, o mais recente foi:

```text
10.0.0.1 - - [06/Jun/2024:13:54:44] "GET /contact HTTP/1.1" 500
```

#### Identificar a última requisição POST de um IP específico

```bash
grep "172.16.0.1" access.log | grep "POST"
```

- Primeiro `grep`: filtra linhas pelo endereço informado.
- `|` (pipe): envia a saída do primeiro comando ao segundo.
- Segundo `grep`: seleciona linhas que contêm POST.

Foram retornados dez registros. O mais recente foi:

```text
172.16.0.1 - - [06/Jun/2024:13:55:44] "POST /contact HTTP/1.1" 500
```

| Campo | GET mais recente | POST mais recente |
|---|---|---|
| IP | 10.0.0.1 | 172.16.0.1 |
| Timestamp | 06/Jun/2024:13:54:44 | 06/Jun/2024:13:55:44 |
| Recurso | /contact | /contact |
| Protocolo | HTTP/1.1 | HTTP/1.1 |
| Status | 500 | 500 |

O código HTTP 500 indica erro interno do servidor. Não comprova, por si só, exploração de uma vulnerabilidade.

**Melhoria técnica:** o comando usado no exercício funcionou sobre os dados fornecidos, mas o ponto é um metacaractere em expressões regulares. Em uma investigação real, pode-se usar `grep -F` para uma busca literal e delimitar o campo de origem para evitar correspondências indevidas. Exemplo de alternativa, não executada no laboratório:

```bash
grep -F '172.16.0.1 - - [' access.log | grep -F '"POST '
```

Também é importante verificar a ordem cronológica pelos timestamps, sem presumir que a primeira ou última linha do arquivo seja sempre o evento mais recente.

## 6. Competências e próximos passos

### Competências desenvolvidas

- Triagem de alertas com metodologia 5 Ws e contextualização de atividades autorizadas.
- Interpretação de registros de tráfego e identificação de comunicações de retorno.
- Extração de metadados com `pdfinfo` e `exiftool`.
- Compreensão de preservação de evidências, cadeia de custódia e aquisição forense.
- Classificação de incidentes e diferenciação entre download e execução de malware.
- Conhecimento das fases PICERL, playbooks, SIEM e EDR.
- Filtragem de eventos Windows e construção de linha do tempo.
- Pesquisa de logs HTTP com `grep` e pipe.

### Conclusão

O módulo consolidou a transição de fundamentos de segurança para atividades defensivas. A prática de SOC L1 passou a envolver a identificação de evidências, validação de contexto, correlação de registros e documentação de resultados, em vez de apenas executar técnicas ofensivas.

O próximo passo é aprofundar a trilha SOC Level 1, com exercícios de investigação de alertas, análise de logs, SIEM, EDR e escalonamento de incidentes. Os conceitos foram praticados em laboratórios guiados; o objetivo seguinte é ganhar autonomia por meio de repetição em novos cenários.

---

**Escopo de publicação:** relatório educacional baseado nas evidências registradas durante os exercícios. Flags, senhas e endereços temporários das máquinas de laboratório foram omitidos. Os IPs privados do cenário de investigação foram mantidos por serem evidências didáticas necessárias à compreensão dos logs.
