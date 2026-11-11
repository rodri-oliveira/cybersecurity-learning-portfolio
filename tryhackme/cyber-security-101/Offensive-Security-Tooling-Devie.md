# TryHackMe — Offensive Security Tooling | Devie

> **Laboratório prático de segurança ofensiva em ambiente autorizado**  
> **Plataforma:** TryHackMe  
> **Contexto:** Offensive Security Tooling  
> **Máquina:** Devie  
> **Status:** Sala concluída  
> **Data:** Novembro de 2026

## 1. Visão geral

Este relatório documenta a análise e a exploração controlada de uma aplicação web Python em um laboratório do TryHackMe. O exercício envolveu reconhecimento de serviços, revisão de código-fonte disponibilizado pela aplicação, identificação de execução insegura de expressões, obtenção de uma sessão remota, análise de permissões Linux e investigação de um mecanismo de backup que seguia links simbólicos.

**Escopo e ética:** todas as atividades descritas foram realizadas na máquina disponibilizada para treinamento. Não são publicadas flags, senhas, chaves de criptografia ou respostas de desafios. Endereços de IP usados durante a sessão foram substituídos por marcadores genéricos.

### Objetivos de aprendizagem

- Identificar serviços expostos e aplicações web relevantes.
- Correlacionar comportamento da aplicação e vulnerabilidades de código.
- Entender como o uso de `eval()` com entrada não confiável pode levar à execução de comandos.
- Trabalhar com shells reversas e suas limitações de interatividade.
- Interpretar permissões de arquivos e regras específicas de `sudo`.
- Compreender o papel de XOR e Base64 em um mecanismo simples de codificação.
- Investigar riscos associados a tarefas privilegiadas que copiam arquivos seguindo links simbólicos.

## 2. Ambiente e ferramentas

| Componente | Utilização |
|---|---|
| TryHackMe AttackBox | Terminal para reconhecimento, requisições HTTP e recepção da shell. |
| Nmap | Identificação dos serviços TCP expostos. |
| `curl` | Envio e inspeção de requisições HTTP. |
| `wget` e `unzip` | Recuperação e análise do código-fonte público da aplicação. |
| Netcat (`nc`) | Recepção de conexões durante a exploração controlada. |
| Python 3 | Análise da aplicação e melhoria da interatividade da shell. |
| `sudo`, `su`, `id`, `ls`, `cat` | Enumeração de identidade, autorizações e permissões. |
| `find`, `getcap`, `ps` | Enumeração de arquivos e processos durante a investigação. |

### Serviços identificados

| Porta TCP | Serviço observado | Relevância |
|---|---|---|
| `22` | OpenSSH | Superfície de acesso remoto identificada no reconhecimento. |
| `5000` | Aplicação HTTP Python/Flask, servida por Werkzeug | Superfície utilizada para identificar a falha inicial. |

A aplicação disponibilizava funcionalidades matemáticas, incluindo cálculos por bisseção.

## 3. Reconhecimento e descoberta do código-fonte

O reconhecimento inicial foi feito com:

```bash
nmap -sV <IP_ALVO>
```

- `nmap`: verifica portas e serviços acessíveis.
- `-sV`: tenta identificar o serviço e sua versão.
- `<IP_ALVO>`: endereço da máquina do laboratório.

A inspeção da aplicação na porta `5000` revelou um arquivo ZIP acessível em um diretório estático:

```bash
wget http://<IP_ALVO>:5000/static/source.zip
unzip -l source.zip
unzip -p source.zip math/app.py
unzip -p source.zip math/bisection.py
```

Os comandos permitiram listar o conteúdo do pacote e ler arquivos específicos sem depender da interface gráfica.

**Descoberta:** a aplicação recebia dados de formulários como strings, e uma das rotinas avaliava diretamente uma expressão construída a partir desses dados.

## 4. Vulnerabilidade principal — uso inseguro de `eval()`

No código analisado, a lógica de bisseção continha a seguinte estrutura:

```python
added = xa + " + " + xb
c = eval(added)
c = int(c) / 2
```

A função `eval()` avalia uma expressão Python. Quando a expressão inclui valores controlados por um usuário sem validação adequada, esse comportamento pode ultrapassar o objetivo matemático original e permitir a execução de instruções não previstas.

O campo `xb` era tratado como texto pelo formulário, criando uma condição favorável à injeção de expressão Python. A exploração controlada demonstrou que a avaliação ocorria **antes** de uma conversão numérica posterior, tornando possível observar execução de código mesmo quando a requisição terminava em erro HTTP `500`.

### Evidência de impacto

Foi estabelecida uma conexão de retorno para um listener na AttackBox, resultando em uma sessão Bash executada sob a identidade do usuário da aplicação:

```text
bruce@<HOST_LAB>:~$
```

A conexão foi recebida por um listener Netcat:

```bash
nc -lvnp 4444
```

- `-l`: modo de escuta (*listen*).
- `-v`: saída detalhada.
- `-n`: evita resolução de nomes.
- `-p 4444`: utiliza a porta local especificada.

**Aprendizado:** a mensagem HTTP de erro, isoladamente, não é evidência de que uma tentativa falhou. É necessário avaliar também seus efeitos observáveis, como conexões e processos criados.

### Classificação técnica

- **Categoria:** injeção de código no servidor.
- **Causa raiz:** avaliação de entrada não confiável com `eval()`.
- **Referência conceitual:** CWE-94 — *Improper Control of Generation of Code*.
- **Impacto observado:** execução de comandos com as permissões do processo da aplicação (`bruce`).

## 5. Pós-exploração: identidade, permissões e primeira flag

Com a shell ativa, a investigação concentrou-se em arquivos associados ao desafio e na identidade efetiva do processo.

O comando:

```bash
id
```

mostrou que a sessão pertencia ao usuário `bruce`, sem participação no grupo `gordon`.

A análise de permissões do segundo arquivo de desafio apresentou:

```text
-rw-r----- 1 root gordon ... /home/gordon/flag2.txt
```

Interpretação:

| Campo | Significado |
|---|---|
| `rw-` | O proprietário `root` pode ler e escrever. |
| `r--` | Integrantes do grupo `gordon` podem ler. |
| `---` | Outros usuários não podem acessar o arquivo. |

**Resultado:** a primeira flag foi recuperada na sessão de `bruce`, enquanto a segunda exigia uma identidade ou autorização adicional. O valor da flag não é reproduzido neste relatório.

## 6. Transição de `bruce` para `gordon`

A verificação das autorizações de `sudo` revelou uma regra restrita:

```text
(gordon) NOPASSWD: /usr/bin/python3 /opt/encrypt.py
```

Essa regra permitia executar **somente o comando especificado** como `gordon`, sem solicitar a senha de `sudo`. Ela não significava permissão irrestrita para executar qualquer comando como esse usuário.

### Script de criptografia e pista do laboratório

A leitura direta de `/opt/encrypt.py` como `bruce` retornava `Permission denied`, mas sua execução como `gordon` era autorizada. O arquivo `/home/bruce/note` indicava que Gordon utilizava uma transformação baseada em **XOR**, com uma representação final em **Base64**.

Conceitualmente:

- **XOR** é uma operação bit a bit reversível quando a mesma chave é conhecida.
- **Base64** é uma codificação de dados, **não** um algoritmo de criptografia.
- Uma transformação reversível com chave reutilizada pode expor informações quando há pares conhecidos de entrada e saída.

O laboratório permitiu testar a rotina e compreender a relação entre entrada conhecida, transformação XOR e resultado codificado. As credenciais e os valores criptográficos concretos foram omitidos.

Para tornar a shell mais adequada a comandos interativos, foi utilizado:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

- `-c`: executa o trecho Python informado na linha de comando.
- `pty`: módulo para criação de pseudoterminais.
- `pty.spawn("/bin/bash")`: inicia um Bash associado ao pseudoterminal.

Em seguida, a autenticação com `su gordon` foi concluída, confirmada pelo prompt:

```text
gordon@<HOST_LAB>:$
```

**Resultado:** a segunda flag foi recuperada com as permissões de `gordon`. Seu conteúdo não é publicado.

## 7. Investigação da etapa de privilégios elevados

Com acesso como `gordon`, foram realizadas verificações adicionais:

```bash
find / -type f -perm -4000 2>/dev/null
getcap -r / 2>/dev/null
cat /etc/crontab
ps -ef --forest
```

O objetivo era identificar binários SUID, capacidades especiais, agendamentos e processos relevantes. Essas verificações **não evidenciaram, por si só, um caminho direto para uma shell de `root`**.

O ponto mais relevante surgiu da comparação dos diretórios:

```text
/home/gordon/reports/
/home/gordon/backups/
```

Foram observados arquivos de relatórios com cópias recentes em `backups` e propriedade de `root`, indicando uma rotina de cópia com privilégios elevados. O mecanismo completo dessa rotina não foi inspecionado diretamente durante a sessão registrada.

### Teste controlado de link simbólico

Para investigar como o backup lidava com links simbólicos, foi criado um novo relatório apontando para um arquivo não sensível do sistema:

```bash
ln -s /etc/hostname /home/gordon/reports/report4
```

- `ln`: cria links entre caminhos.
- `-s`: cria um **link simbólico** (*symlink*).
- `/etc/hostname`: arquivo de teste escolhido por ser inofensivo.
- `report4`: nome do link criado no diretório de entrada.

Depois da atualização do backup, foi observado:

```text
/home/gordon/backups/report4
```

O arquivo copiado tinha propriedade de `root` e seu conteúdo correspondia ao nome do host presente em `/etc/hostname`.

**Conclusão demonstrada:** a rotina de backup seguiu o link simbólico e materializou no diretório de saída o conteúdo do arquivo apontado por ele. Esse comportamento pode causar exposição de arquivos quando um processo privilegiado lê caminhos controlados por usuários com menos privilégios.

Posteriormente, o link foi redirecionado para um caminho suposto de arquivo protegido. **A saída registrada no terminal ainda mostrava o conteúdo do backup anterior**, portanto não há evidência nessa sessão de que o conteúdo protegido tenha sido copiado. Também não foi demonstrada uma sessão interativa de `root`.

Essa distinção é importante para manter a documentação técnica fiel às evidências obtidas.

### Classificação técnica

- **Categoria:** tratamento inseguro de links simbólicos em operações de arquivo.
- **Referência conceitual:** CWE-59 — *Improper Link Resolution Before File Access*.
- **Impacto potencial:** leitura ou cópia de arquivos fora do diretório pretendido por uma rotina privilegiada.
- **Impacto comprovado no laboratório:** acompanhamento de um link simbólico em operação de backup, inicialmente com um arquivo de teste não sensível.

## 8. Linha do tempo dos resultados

| Etapa | Evidência | Resultado |
|---|---|---|
| Reconhecimento | Serviços nas portas `22` e `5000` | Superfície web identificada. |
| Inspeção da aplicação | Pacote `source.zip` e código Python | Uso inseguro de `eval()` localizado. |
| Execução remota | Conexão reversa recebida | Shell como `bruce`. |
| Permissões Linux | Identidade e grupos consultados | Restrições sobre arquivo de `gordon` confirmadas. |
| Regra `sudo` | Execução autorizada de `encrypt.py` | Pista para transição de usuário. |
| Autenticação | Prompt de `gordon` | Sessão como `gordon`; segunda flag recuperada. |
| Investigação de backups | Arquivos em `reports` e `backups` | Rotina privilegiada identificada por efeitos observáveis. |
| Link simbólico | `report4` apontando para `/etc/hostname` | Comportamento de seguir symlink confirmado. |
| Conclusão da sala | Registro de conclusão do exercício | Sala finalizada; valores das flags omitidos. |

## 9. Recomendações defensivas

**Para a aplicação Python:**

1. Substituir o uso de `eval()` por cálculos explícitos ou por um avaliador matemático restrito, quando estritamente necessário.
2. Validar entradas com tipos e formatos esperados antes de qualquer processamento.
3. Executar a aplicação com uma conta de serviço de privilégios mínimos.
4. Registrar e monitorar falhas inesperadas, sem expor detalhes internos aos clientes.
5. Evitar publicar pacotes de código-fonte e arquivos desnecessários em diretórios estáticos.

**Para identidade e sistema operacional:**

1. Revisar regras `sudoers` e scripts executados em nome de outros usuários.
2. Evitar mecanismos próprios de proteção de senhas com XOR e Base64; usar abordagens apropriadas ao propósito (hash de senha com função de derivação adequada, ou armazenamento seguro de segredos quando reversibilidade for exigida).
3. Proteger permissões de diretórios e arquivos utilizados por serviços privilegiados.
4. Impedir que rotinas de backup sigam links simbólicos fornecidos por fontes não confiáveis, além de validar caminhos e propriedades dos arquivos.
5. Separar claramente diretórios de entrada controlados por usuários e diretórios usados em operações privilegiadas.

## 10. Aprendizados consolidados

- **Reconhecimento não é exploração:** identificar um serviço é apenas o início; a exploração exige compreensão de seu comportamento.
- **Código-fonte pode esclarecer uma falha:** a revisão de `app.py` e `bisection.py` foi determinante para localizar o risco do `eval()`.
- **Permissões devem ser interpretadas por proprietário, grupo e outros:** localizar um arquivo não significa poder lê-lo.
- **Regras `sudo` precisam ser analisadas literalmente:** uma autorização para executar um script não equivale a acesso irrestrito.
- **Codificação não é criptografia:** Base64 apenas representa dados; XOR com chave reutilizada pode oferecer proteção inadequada.
- **Processos privilegiados exigem validação rigorosa de caminhos:** links simbólicos podem desviar uma operação para arquivos não previstos.
- **Evidência e hipótese devem ser separadas:** um comportamento confirmado em um teste inofensivo não prova automaticamente a leitura de qualquer outro arquivo.

## 11. Considerações finais

O laboratório reuniu uma cadeia de atividades importante para a formação em segurança: reconhecimento, análise de código, exploração de aplicação web, obtenção de shell, movimentação entre identidades Linux e investigação de riscos em operações de backup privilegiadas.

A experiência também reforçou a disciplina de documentar **o que foi efetivamente observado**, distinguir resultados de hipóteses e preservar credenciais e flags fora de repositórios públicos.

---

**Uso:** portfólio educacional em segurança da informação / TryHackMe.  
**Aviso:** conteúdo destinado exclusivamente a ambientes próprios, autorizados ou de laboratório.
