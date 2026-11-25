# Defensive Security Tooling — TryHackMe

> Cyber Security 101 | Portfólio de Segurança da Informação | Novembro de 2026

## Sumário

- [Objetivos](#objetivos)
- [CyberChef — The Basics](#cyberchef--the-basics)
- [CAPA — The Basics](#capa--the-basics)
- [REMnux — Getting Started](#remnux--getting-started)
- [FLARE-VM — Arsenal of Tools](#flare-vm--arsenal-of-tools)
- [Indicadores de comprometimento](#indicadores-de-comprometimento)
- [Limitações e lições aprendidas](#limitações-e-lições-aprendidas)
- [Competências desenvolvidas](#competências-desenvolvidas)

## Objetivos

Documentar atividades de análise estática, triagem de executáveis, investigação de memória e simulação de serviços de rede em laboratórios autorizados do TryHackMe. O foco é desenvolver competências aplicáveis à atuação como SOC Analyst L1, distinguindo evidências observadas, hipóteses e resultados apresentados pelo material didático.

O relatório não contém flags, senhas, amostras de malware ou scripts maliciosos completos. Os endereços externos suspeitos estão neutralizados (*defanged*). Os endereços privados de laboratório são temporários.

## CyberChef — The Basics

**Objetivo:** transformar e interpretar dados utilizando operações encadeadas (*recipes*), sem executar seu conteúdo.

Foram estudadas operações de codificação e decodificação, representação binária e transformação de datas. Entre os resultados registrados estão a sequência binária `01001110`, a representação Base64 `TmljZSBSb29tIQ==` e a data `Sun 1 September 2024 00:40:58 UTC`. A operação `From Base85` também foi abordada; a sequência original não foi preservada no registro disponível.

A aplicação prática mais relevante para este módulo foi a desofuscação de uma macro VBA na sala REMnux, descrita adiante. Codificação não equivale a criptografia, e a recuperação de texto não implica execução do código.

## CAPA — The Basics

**Objetivo:** identificar capacidades de executáveis por meio de análise estática baseada em regras e relacioná-las a técnicas de ameaças.

### Procedimentos registrados

```powershell
Get-FileHash .\cryptbot.bin -Algorithm SHA256
capa.exe -j .\cryptbot.bin > cryptbot.json
```

- `Get-FileHash`: calcula o hash do arquivo; `-Algorithm SHA256` seleciona o algoritmo.
- `capa.exe`: ferramenta de identificação de capacidades; `-j` solicita saída JSON; `>` salva a saída em um arquivo.
- O CAPA Web Explorer e sua caixa de pesquisa global foram utilizados para interpretar os resultados.

**SHA-256 registrado:** `AE7BC6B6F6ECB206A7B957E4BB86E0D11845C5B2D9F7A00A482BEF63B567CE4C`

Foram identificadas referências às técnicas MITRE ATT&CK `T1027` (Obfuscated Files or Information) e `T1027.005` (Indicator Removal from Tools), além das categorias MBC relacionadas a *Launcher* e *Downloader* e da função `RegOpenKeyEx`.

Essas correspondências indicam capacidades ou características detectadas pelas regras; não comprovam que todas tenham sido executadas. Os detalhes completos das saídas dessa sala não estão disponíveis neste registro, portanto não são reconstruídos por suposição.

## REMnux — Getting Started

**Ambiente:** REMnux (Linux), AttackBox e ferramentas pré-instaladas. Atividades registradas em 24/11/2026.

### 1. Análise estática de documentos Office

```bash
cd /home/ubuntu/Desktop/tasks/agenttesla/
oledump.py agenttesla.xlsm
oledump.py agenttesla.xlsm -s 4
oledump.py agenttesla.xlsm -s 4 --vbadecompress
oledump.py possible_malicious.docx
```

| Elemento | Finalidade |
|---|---|
| `oledump.py` | Inspecionar estruturas OLE e fluxos de dados |
| `-s 4` | Selecionar o fluxo 4 |
| `--vbadecompress` | Descompactar o código VBA |
| `cd` | Alterar o diretório de trabalho |

**Resultados:**

| Amostra | Fluxos | Macro identificada | Tamanho do fluxo |
|---|---:|---|---:|
| `agenttesla.xlsm` | 6 | A4 — `VBA/ThisWorkbook` | 688 bytes |
| `possible_malicious.docx` | 16 | 8 — `Macros/VBA/ThisDocument` | 1.989.192 bytes |

O marcador `M` identifica código VBA; `m` indica um fluxo contendo apenas atributos de macro. O segundo documento apresentava estrutura OLE binária apesar da extensão `.docx`.

No Excel, foram observados o evento `Workbook_Open`, a variável `Sqtnew`, chamadas `Replace`, a criação de `WScript.Shell` e a chamada `Exec`. O CyberChef foi utilizado com duas operações `Find / Replace`, em modo `SIMPLE STRING`, para remover os caracteres de ofuscação conforme a lógica da macro.

A análise revelou referências ao PowerShell, execução em janela oculta, política de execução `bypass`, criação de arquivo temporário, `Invoke-WebRequest` e `Start-Process`. A intenção inferida era baixar e iniciar um executável remoto. O script apresentava um caractere de substituição de codificação; não foi reparado nem executado.

**Conclusão:** evidências estáticas fortemente suspeitas, sem confirmação de download ou execução real da ameaça.

### 2. Rede simulada — INetSim

**Topologia:** AttackBox `10.66.82.52` → REMnux `10.66.136.23`.

```bash
ifconfig
grep -n dns_default_ip /etc/inetsim/inetsim.conf
sudo nano +207 /etc/inetsim/inetsim.conf
cat /etc/inetsim/inetsim.conf | grep dns_default_ip
sudo inetsim
```

A configuração `dns_default_ip 10.66.136.23` foi ativada e validada. O HTTPS (443) iniciou corretamente. O HTTP (80) falhou, houve conflito de socket UDP no DNS e um aviso de opção desconhecida na linha 43. Essas falhas não foram corrigidas, pois o exercício HTTPS funcionou.

O Firefox apresentou `MOZILLA_PKIX_ERROR_SELF_SIGNED_CERT`, compatível com o certificado autoassinado do servidor simulado. A exceção foi aceita exclusivamente dentro do laboratório.

Na AttackBox, foram realizadas solicitações HTTPS de arquivos simulados:

```bash
sudo wget https://10.66.136.23/second_payload.zip --no-check-certificate
sudo wget https://10.66.136.23/second_payload.ps1 --no-check-certificate
ls -lh
```

O parâmetro `--no-check-certificate` foi usado apenas no ambiente controlado. Os dois downloads retornaram HTTP `200 OK`, 258 bytes e `text/html`, evidenciando que a extensão do arquivo não determina seu conteúdo. Nenhuma amostra real foi executada.

Após encerrar o INetSim com `Ctrl+C`, foram analisados os relatórios:

```bash
sudo cat /var/log/inetsim/report/report.2286.txt
sudo cat /var/log/inetsim/report/report.2345.txt
```

| Sessão | Horário do servidor | Método | Recurso solicitado | Arquivo fornecido |
|---|---|---|---|---|
| 2286 | 14:17:10 | GET | `/` | `sample.html` |
| 2286 | 14:17:10 | GET | `/favicon.ico` | `favicon.ico` |
| 2286 | 14:19:03 | GET | `/second_payload.zip` | `sample.html` |
| 2286 | 14:19:36 | GET | `/second_payload.ps1` | `sample.html` |
| 2345 | 14:23:24 | GET | Arquivo textual do desafio | `sample.txt` |

A última solicitação retornou 151 bytes e `text/plain`. O conteúdo da flag foi omitido. A correlação dos registros do cliente e do servidor permitiu reconstruir a sequência de eventos; pequenas diferenças de um segundo nos horários foram observadas.

**Erros corrigidos:** o atalho `Ctrl+W` foi interceptado pelo navegador; a localização da configuração foi realizada com `grep -n` e abertura direta na linha 207. Uma leitura de relatório falhou por ter sido executada na AttackBox, em vez do REMnux; o comando foi repetido na máquina correta.

### 3. Forense de memória — Volatility 3

**Amostra:** `wcry.mem` | **Versão:** Volatility 3 Framework 2.0.0.

```bash
sudo su
cd /home/ubuntu/Desktop/tasks/Wcry_memory_image/
vol3 -f wcry.mem windows.pstree.PsTree
vol3 -f wcry.mem windows.pslist.PsList
vol3 -f wcry.mem windows.cmdline.CmdLine
```

`-f` especifica a imagem de memória. Os plugins `PsTree`, `PsList` e `CmdLine` permitem observar hierarquia, processos ativos e argumentos, respectivamente.

**Resultado:** 20 processos identificados tanto em PsTree quanto em PsList.

```text
explorer.exe (1636)
└── tasksche.exe (1940)
    └── @WanaDecryptor@ (740)
```

| Processo | PID | PPID | Criação (12/05/2017) |
|---|---:|---:|---|
| explorer.exe | 1636 | 1608 | 21:22:10 |
| tasksche.exe | 1940 | 1636 | 21:22:14 |
| @WanaDecryptor@ | 740 | 1940 | 21:22:22 |

O `CmdLine` recuperou o caminho `C:\Intel\ivecuqmanpnirkt615\tasksche.exe`. O processo `System` (PID 4) apresentou erro de acesso a uma região de memória ao recuperar seus argumentos, sem evidência suficiente para concluir que havia sido encerrado.

#### Pré-processamento das evidências

```bash
vol3 -f wcry.mem windows.filescan.FileScan > wcry.filescan.FileScan.txt
vol3 -f wcry.mem windows.dlllist.DllList > wcry.dlllist.DllList.txt
vol3 -f wcry.mem windows.malfind.Malfind > wcry.malfind.Malfind.txt
vol3 -f wcry.mem windows.psscan.PsScan > wcry.psscan.PsScan.txt
vol3 -f wcry.mem windows.pstree.PsTree > wcry.pstree.PsTree.txt
vol3 -f wcry.mem windows.pslist.PsList > wcry.pslist.PsList.txt
vol3 -f wcry.mem windows.cmdline.CmdLine > wcry.cmdline.CmdLine.txt
strings wcry.mem > wcry.strings.ascii.txt
strings -e l wcry.mem > wcry.strings.unicode_little_endian.txt
strings -e b wcry.mem > wcry.strings.unicode_big_endian.txt
ls -lh *.txt
```

`>` redireciona a saída padrão para um arquivo; `-e l` e `-e b` selecionam respectivamente caracteres de 16 bits little-endian e big-endian; `ls -lh` lista os arquivos com tamanhos legíveis.

| Artefato | Tamanho observado |
|---|---:|
| CmdLine | 1,4 K |
| DllList | 81 K |
| FileScan | 69 K |
| Malfind | 14 K |
| PsList | 1,8 K |
| PsScan | 2,2 K |
| PsTree | 1,7 K |
| Strings ASCII | 12 M |
| Strings Big Endian | 6,2 M |
| Strings Little Endian | 6,3 M |

Os dez arquivos foram confirmados como não vazios. A sala também apresentou automação com laço `for`; o procedimento efetivamente realizado foi individual, com redirecionamento de saída.

#### Investigação das regiões suspeitas

```bash
head -n 30 wcry.malfind.Malfind.txt
grep -i '@WanaDecryptor@.exe' wcry.dlllist.DllList.txt
```

`head -n 30` exibe as primeiras 30 linhas; `grep -i` pesquisa sem distinção de maiúsculas e minúsculas. O padrão usado no grep foi o registrado no laboratório; para uma busca literal, `grep -Fi` seria mais preciso.

| PID | Processo | Região inicial | Região final | Proteção |
|---|---|---|---|---|
| 596 | csrss.exe | 0x7f6f0000 | 0x7f7effff | PAGE_EXECUTE_READWRITE |
| 620 | winlogon.exe | 0x21400000 | 0x21403fff | PAGE_EXECUTE_READWRITE |

As regiões RWX (leitura, escrita e execução) foram sinalizadas pelo Malfind. A presença de permissões RWX é um indicador para investigação, não confirmação de injeção maliciosa.

O DllList confirmou o caminho `C:\Intel\ivecuqmanpnirkt615\@WanaDecryptor@.exe`, base `0x400000`, tamanho `0x3d000`. A correlação com PsTree e CmdLine reforça a associação entre os executáveis da amostra WannaCry.

O PsScan e FileScan foram executados e preservados, mas não foi realizada uma comparação exaustiva nem uma investigação completa de seus resultados.

## FLARE-VM — Arsenal of Tools

**Ambiente:** Windows, PEStudio 9.22, CFF Explorer VIII e Sysinternals. Atividades registradas em 25/11/2026.

### 1. Ferramentas estudadas

| Ferramenta | Aplicação |
|---|---|
| x64dbg / Ghidra | Depuração e engenharia reversa |
| Process Hacker / Process Explorer | Investigação de processos e memória |
| Procmon | Registro de atividades do sistema e rede |
| FTK Imager / Volatility | Aquisição de disco e forense de memória |
| HxD | Inspeção hexadecimal |
| CFF Explorer / PEStudio | Estruturas PE, metadados, hashes e indicadores |
| FLOSS | Extração e recuperação de strings |
| Wireshark | Análise de tráfego de rede |

A relação `System` (PID 4) → `smss.exe` (PID 288) foi confirmada localmente no Process Explorer, estabelecendo uma referência de comportamento normal do Windows.

### 2. Análise estática com PEStudio

| Propriedade | cryptominer.bin / windows.exe | cobaltstrike.exe |
|---|---|---|
| Arquitetura | 64 bits | 64 bits |
| Tamanho | 2.004.480 bytes | 17.920 bytes |
| Entropia | 7.999 | 5.206 |
| Tecnologia | Microsoft .NET | Não indicada |
| Descrição | REGEDIT | N/A |
| Imphash | N/A | `92EEF189FB188C541CBD83AC8BA4ACF5` |

`cryptominer.bin` e `windows.exe` apresentaram o mesmo SHA-256, indicando conteúdo binário idêntico com elevada confiança, apesar dos nomes diferentes.

**SHA-256:** `E9627EBAAC562067759681DCEBA8DDE8D83B1D813AF8181948C549E342F67C0E`

Foram observados cabeçalho MZ, ausência de Rich Header, descrição REGEDIT, alta entropia e manifesto contendo `requestedExecutionLevel` com valor `requireAdministrator` e `uiAccess=false`.

A consulta VirusTotal foi desabilitada na configuração do PEStudio após erro de comunicação e demora no processamento. A listagem de `cryptominer.bin` apresentou 102 funções e uma sinalização de blacklist. Não foi comprovado que a consulta externa causava a demora.

O material didático de `windows.exe` destacou `set_UseShellExecute`, `RijndaelManaged`, `CryptoStream`, `CipherMode` e `CreateDecryptor`. Esses nomes indicam possíveis capacidades de execução de processos e criptografia, mas não demonstram seu uso efetivo. A execução do FLOSS para essa amostra foi apresentada pela sala, não registrada como realizada localmente.

### 3. Análise com CFF Explorer

A amostra `possible_medusa.txt` foi aberta no CFF Explorer sem execução.

| Propriedade | Resultado |
|---|---|
| Formato real | Portable Executable 32 |
| Tamanho | 685.568 bytes |
| Compilador indicado | Microsoft Visual C++ 8 |
| MD5 | `646698572AFBBF24F50EC5681FEB2DB7` |
| SHA-1 | `70530BC23BAD38E6AEE66CBB2C2F58A96A18FB79` |

A divergência entre extensão `.txt` e formato PE32 foi confirmada. A questão sobre o campo `e_magic` foi abordada, mas não há captura local do valor desse campo no registro disponível.

### 4. Investigação de processos e rede

No Procmon, foi aplicado o filtro `Process Name contains cobalt` com ação `Include`. Nenhum evento correspondente apareceu. O Process Explorer também não apresentou o processo ativo; a árvore do Procmon não revelou o executável na captura inspecionada.

Para responder às questões, foram utilizadas as capturas fornecidas pelo TryHackMe:

| Campo | Evidência didática |
|---|---|
| Processo pai | explorer.exe (PID 1208) |
| Processo filho | cobaltstrike.exe (PID 4756) |
| Destino | `47[.]120[.]46[.]210` |
| Porta | TCP 81 (`hosts2-ns`) |
| Estado | SYN_SENT |

`SYN_SENT` indica uma tentativa de estabelecimento de conexão TCP, não uma sessão confirmada de comando e controle. Os PIDs das diferentes capturas didáticas podem variar. A amostra não foi executada durante o procedimento acompanhado; a comunicação e o processo pai não foram reproduzidos localmente.

## Indicadores de comprometimento

Indicadores observados nas amostras ou apresentados pelo material didático; não representam confirmação de incidentes em sistemas reais.

| Indicador | Origem | Tipo |
|---|---|---|
| `193[.]203[.]203[.]67` | Macro Agent Tesla, análise local | IP remoto referenciado |
| `Doc-3737122pdf.exe` | Macro Agent Tesla, análise local | Nome de arquivo remoto |
| `C:\Intel\ivecuqmanpnirkt615\tasksche.exe` | Volatility CmdLine | Caminho de executável |
| `C:\Intel\ivecuqmanpnirkt615\@WanaDecryptor@.exe` | Volatility DllList | Caminho de executável |
| `47[.]120[.]46[.]210:81` | Captura didática FLARE-VM | Destino de tentativa TCP |
| `92EEF189FB188C541CBD83AC8BA4ACF5` | PEStudio, análise local | Imphash |

## Limitações e lições aprendidas

- As evidências locais não foram confundidas com exemplos fornecidos pela plataforma.
- Alta entropia, nomes suspeitos, funções criptográficas e regiões RWX exigem contextualização; isoladamente, não comprovam atividade maliciosa.
- Hashes idênticos podem relacionar arquivos renomeados; MD5 e SHA-1 não devem ser o único mecanismo de verificação contra adulterações intencionais.
- O INetSim apresentou falhas de DNS/HTTP, mas o serviço HTTPS foi validado por requisições e relatórios.
- O relatório não afirma aquisição forense original, cadeia de custódia formal, contenção real ou execução de malware, pois esses procedimentos não foram realizados.
- Os registros de CyberChef e CAPA disponíveis são menos detalhados que os das salas REMnux e FLARE-VM; não foram inventados resultados adicionais.
- As salas Dig Dug e Training Impact on Teams não foram incluídas, por serem atividades distintas do módulo aqui documentado.

## Competências desenvolvidas

- Triagem estática de documentos Office e executáveis Windows.
- Identificação e interpretação de macros VBA e strings ofuscadas.
- Análise de hashes, metadados, entropia, importações e manifesto PE.
- Simulação de serviços de rede e correlação de registros HTTPS.
- Pré-processamento de imagens de memória com Volatility 3 e GNU strings.
- Investigação de hierarquia de processos, argumentos e módulos carregados.
- Identificação, neutralização e documentação de potenciais IOCs.
- Comunicação técnica de hipóteses, evidências e limitações, com foco em SOC L1.

## Referências

- [TryHackMe](https://tryhackme.com/) — Cyber Security 101, Defensive Security Tooling.
- [CyberChef](https://gchq.github.io/CyberChef/)
- [REMnux](https://remnux.org/)
- [FLARE-VM](https://github.com/mandiant/flare-vm)
- [CAPA](https://github.com/mandiant/capa)
- [Volatility 3](https://github.com/volatilityfoundation/volatility3)
- [Sysinternals](https://learn.microsoft.com/en-us/sysinternals/)

*Relatório educacional baseado em atividades autorizadas e registros do laboratório. Não contém amostras executáveis nem respostas secretas de desafios.*
