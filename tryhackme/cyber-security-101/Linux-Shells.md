# Linux Shells — TryHackMe | Relatório técnico de laboratório

> **Trilha:** Cyber Security 101 · **Plataforma:** TryHackMe  
> **Ambiente:** máquina Linux/Ubuntu disponibilizada pelo laboratório, acesso via terminal  
> **Autor:** [rodri-oliveira](https://github.com/rodri-oliveira)  
> **Foco:** fundamentos de shells, Bash, automação, permissões e investigação de arquivos de log.

## 1. Objetivo e abordagem

Este relatório registra os conhecimentos e **os testes efetivamente realizados** durante a sala *Linux Shells*, com atenção à compreensão da sintaxe, à investigação de erros e às aplicações em Segurança da Informação. Em vez de somente executar comandos para concluir as tarefas, o laboratório foi conduzido passo a passo: observar a saída, compreender o comportamento do shell, corrigir o problema e retestar.

**Nota de ética:** não são publicadas flags, credenciais nem respostas diretas dos desafios. Caminhos de laboratório e exemplos ilustrativos são utilizados quando necessários para explicar o método.

## 2. Fundamentos de shell

O **shell** é a interface que recebe comandos e os encaminha ao sistema operacional. O **Bash** (*Bourne Again Shell*) é um interpretador de comandos e uma linguagem de scripts amplamente utilizada em Linux. Outros shells possuem recursos próprios, como o **fish**, conhecido por realce de sintaxe e sugestões interativas.

### Comandos explorados

| Comando | O que faz | Observação técnica |
|---|---|---|
| `pwd` | Mostra o diretório de trabalho atual | Ajuda a entender a origem dos caminhos relativos. |
| `ls` | Lista entradas do diretório | Não mostra por padrão nomes ocultos iniciados por ponto. |
| `ls -a` | Lista também entradas ocultas | `-a` significa *all*. |
| `ls -l` | Lista em formato detalhado | Mostra permissões, proprietário, grupo, tamanho e data. |
| `cat arquivo` | Exibe o conteúdo do arquivo | Não executa o texto mostrado. |
| `cat -n arquivo` | Exibe o conteúdo com números de linha | `-n` facilita localizar informações. |
| `grep 'termo' arquivo` | Procura texto em um arquivo | Útil para localizar eventos em registros. |
| `history` | Exibe comandos registrados pelo shell | Útil para revisão; pode conter dados sensíveis dependendo dos comandos digitados. |
| `echo "$0"` | Exibe o nome usado para invocar o shell/script atual | No laboratório, retornou `-bash`, indicando shell de login. |
| `echo "$SHELL"` | Consulta a variável de ambiente `SHELL` | No ambiente observado, não trouxe valor; variáveis de ambiente podem estar ausentes. |

### Leitura dos símbolos

- **`$variavel`**: expansão do valor de uma variável, por exemplo, `$name`.
- **`"texto $variavel"`**: aspas duplas conservam o texto como argumento e permitem expansão de variáveis.
- **`'texto $variavel'`**: aspas simples impedem expansão; a variável aparece literalmente.
- **`>`** em `echo "texto" > arquivo`: redireciona a saída, sobrescrevendo o arquivo.
- **`>>`**: redireciona acrescentando ao final do arquivo.
- **`./script.sh`**: executa um arquivo situado no diretório atual, se houver permissão.
- **`/caminho/absoluto`**: começa na raiz do sistema de arquivos; não depende do diretório atual.
- **`*`**: curinga (*glob*) usado para corresponder a nomes de arquivos.
- **`#`**: inicia comentário no Bash, exceto no uso especial de `#!` na primeira linha do arquivo.
- **`2>/dev/null`**: encaminha mensagens de erro (descritor `2`, *stderr*) para `/dev/null`; recurso discutido nos estudos, mas não usado para esconder erros neste desafio.

## 3. Primeiro script: entrada de dados e variáveis

**Arquivo criado:** `first_script.sh`.

```bash
#!/bin/bash
echo "Hey, what's your name?"
read name
echo "Hello, $name!"
```

**Funcionamento:**

1. `#!/bin/bash` é o *shebang*, que identifica o interpretador do script.
2. `echo` escreve mensagens na saída padrão.
3. `read name` lê uma linha do teclado e guarda o valor na variável `name`.
4. `"Hello, $name!"` utiliza **expansão de variável** dentro de aspas duplas.

**Comandos praticados:**

```bash
nano first_script.sh
chmod +x first_script.sh
ls -l first_script.sh
./first_script.sh
```

- `nano`: editor de texto no terminal; foi utilizado para criar e ajustar scripts.
- `chmod +x`: acrescenta permissão de execução às classes de usuário aplicáveis.
- `ls -l`: permite verificar visualmente a presença do bit `x`.
- `./`: executa um programa do diretório atual.

**Validação observada:** o script perguntou o nome do usuário e imprimiu uma saudação utilizando a resposta digitada.

## 4. Segundo script: automação com repetição

**Arquivo criado:** `loop_script.sh`.

```bash
#!/bin/bash
for i in {1..10}; do
    echo "$i"
done
```

**Análise da sintaxe:**

| Elemento | Função |
|---|---|
| `for` | Inicia um laço de repetição. |
| `i` | Variável temporária que recebe cada valor. |
| `in` | Introduz a sequência percorrida. |
| `{1..10}` | Expansão Bash que gera números de 1 a 10. |
| `; do` | Introduz o bloco de comandos do laço; `;` separa comandos na linha. |
| `echo "$i"` | Imprime o valor atual da variável. |
| `done` | Encerra o laço. |

**Erro real:** na primeira execução, `./loop_script.sh` retornou `Permission denied`. O arquivo não estava habilitado para execução. A correção foi:

```bash
chmod +x loop_script.sh
ls -l loop_script.sh
./loop_script.sh
```

**Resultado:** números de 1 até 10 impressos no terminal. **Lição:** uma sintaxe válida não dispensa as permissões necessárias para executar um script.

## 5. Terceiro script: condicionais e simulação de autenticação

**Arquivo criado:** `locker_script.sh`.

O objetivo didático foi solicitar três informações (usuário, organização e PIN), avaliar condições com `if` e apresentar uma mensagem de autorização ou rejeição. O código abaixo é uma **reconstrução didática**, com valores substituídos para não publicar credenciais do exercício.

```bash
#!/bin/bash
username=""
companyname=""
pin=""

for i in {1..3}; do
    if [ "$i" -eq 1 ]; then
        echo "Enter your Username:"
        read username
    elif [ "$i" -eq 2 ]; then
        echo "Enter your Company:"
        read companyname
    else
        echo "Enter your PIN:"
        read pin
    fi
done

# Credenciais fictícias, apenas para demonstrar as condicionais.
if [ "$username" = "demo_user" ] &&
   [ "$companyname" = "demo_company" ] &&
   [ "$pin" = "0000" ]; then
    echo "Authentication Successful."
else
    echo "Authentication Denied!!"
fi
```

**Conceitos aprendidos:**

- `username=""`, `pin=""`: inicializam variáveis com strings vazias.
- `if`, `elif`, `else`, `fi`: selecionam caminhos de execução conforme uma condição.
- `[ "$i" -eq 1 ]`: comparação **numérica**, `-eq` = *equal*.
- `[ "$username" = "demo_user" ]`: comparação de **strings**; os espaços dentro de `[ ... ]` são necessários à sintaxe do comando de teste.
- `&&`: exige que condições conectadas sejam verdadeiras.
- `read`: coleta entradas de forma interativa.

**Dificuldades reais:** ao editar o código no `nano`, ocorreram problemas de colagem, visualização e organização de quebras de linha. Foram praticados atalhos de salvamento, saída, seleção, corte e desfazer para corrigir o arquivo, sem presumir que toda distorção de tela correspondia a alteração do conteúdo.

**Testes concluídos:** houve uma tentativa com dados incorretos que retornou `Authentication Denied!!` e outra com dados válidos do laboratório que retornou a mensagem de sucesso.

> **Alerta de segurança:** comparar PIN e usuário diretamente com strings codificadas no script **não representa autenticação segura**. Segredos embutidos no código podem ser extraídos; `read` simples também exibe o PIN enquanto ele é digitado. Em sistemas reais, devem ser empregados controles apropriados de identidade, armazenamento seguro de credenciais e mecanismos de auditoria. Este exercício foi exclusivamente educativo.

## 6. Desafio de investigação: busca em arquivos `.log`

**Script disponibilizado no laboratório:** `/home/user/flag_hunt.sh`.

### 6.1 Reconhecimento e leitura prévia

```bash
ls -l /home/user
cat /home/user/flag_hunt.sh
```

Antes de executar um script desconhecido, é importante **ler o conteúdo e entender seu comportamento**. `ls -l` confirmou que o arquivo existia e era executável; `cat` permitiu inspecionar o código sem executá-lo.

### 6.2 Configuração do script

O script possuía campos vazios para o diretório e a palavra pesquisada. Foi configurado para pesquisar os arquivos `.log` de `/var/log`:

```bash
directory="/var/log"
flag="<palavra-chave-do-laboratório>"

for file in "$directory"/*.log; do
    if grep -q "$flag" "$file"; then
        echo "Arquivo com correspondência: $(basename "$file")"
    fi
done
```

O fragmento acima é uma **versão explicativa**, com a palavra-chave e a mensagem substituídas. O arquivo original continha também comentários e uma mensagem de progresso.

**Desmontando os componentes:**

| Código | Explicação |
|---|---|
| `directory="/var/log"` | Define o diretório da investigação. |
| `flag="..."` | Armazena a expressão procurada. |
| `"$directory"/*.log` | Usa o diretório configurado e expande apenas nomes terminados em `.log` **nesse nível** do diretório. |
| `for file in ...; do` | Percorre os arquivos encontrados. |
| `grep -q "$flag" "$file"` | Verifica silenciosamente se há correspondência; `-q` = *quiet*. |
| `if ...; then` | Executa o bloco somente quando `grep` retorna status de sucesso. |
| `$(basename "$file")` | Substituição de comando: extrai apenas o nome do arquivo, removendo o caminho. |
| `fi` e `done` | Encerram, respectivamente, a condicional e o laço. |

**Precisão importante:** esse laço pesquisa arquivos `.log` diretamente em `/var/log`; ele **não é recursivo**. Em um cenário real, isso precisa ser considerado antes de afirmar que todos os arquivos do diretório e subdiretórios foram analisados.

### 6.3 Verificação de sintaxe e execução

```bash
bash -n /home/user/flag_hunt.sh
/home/user/flag_hunt.sh
```

- `bash -n`: lê o script e verifica erros de sintaxe sem executar os comandos.
- A ausência de mensagens após `bash -n` indicou que não foram encontrados erros de sintaxe.
- O script executou e encontrou uma correspondência, mas algumas tentativas de leitura de logs retornaram `Permission denied`.

O laboratório orientava executar a busca com privilégios administrativos para acessar arquivos protegidos. Após conferência do script, foi utilizado:

```bash
sudo /home/user/flag_hunt.sh
```

O segundo teste concluiu sem os erros anteriores de acesso e apontou o arquivo relevante para a atividade.

**Princípio aplicado:** usar privilégios elevados apenas quando necessários e **somente após conferir o que será executado**. O comando `sudo` não deve ser usado como solução genérica para todo erro.

### 6.4 Investigação do arquivo identificado

Para examinar o arquivo apontado pela busca e facilitar a análise da evidência, foi utilizado:

```bash
sudo cat -n /var/log/<arquivo-identificado>.log
```

A opção `-n` numerou as linhas. A leitura permitiu encontrar a informação textual solicitada pela sala, sem necessidade de adivinhar a resposta.

### 6.5 Problemas encontrados e solução

| Ocorrência observada | Causa | Correção / lição |
|---|---|---|
| Conteúdo do script colado diretamente no terminal | Bash tentou interpretar linhas como comandos interativos; apareceu o prompt de continuação `>` | `Ctrl+C` interrompeu a entrada; o conteúdo passou a ser **lido com `cat`** e **editado com `nano`**. |
| `-bash: !/bin/bash: event not found` | A linha do *shebang* foi tratada como entrada interativa e o `!` acionou expansão de histórico | O *shebang* deve permanecer como primeira linha **dentro do arquivo de script**. |
| Busca tentou usar caminho inválido, gerando `grep: ... No such file or directory` | Variável e caminho no laço não estavam configurados corretamente | Preencher `directory` e usar `"$directory"/*.log`. |
| `./home/user/flag_hunt.sh: No such file or directory` | Caminho relativo incorreto: `./home` não equivale a `/home` | Usar `/home/user/flag_hunt.sh` ou, estando no diretório certo, `./flag_hunt.sh`. |
| `grep: ... Permission denied` | Usuário comum não podia ler alguns logs | Depois de inspecionar o código, repetir com `sudo` conforme as instruções do laboratório. |
| Primeira senha do `sudo` foi rejeitada | Credencial digitada não foi aceita | Repetir usando a senha correta da máquina, sem expor credenciais. |

## 7. Outras lições de uso do terminal

### `nano` — atalhos efetivamente praticados

| Atalho | Ação |
|---|---|
| `Ctrl+O` e `Enter` | Gravar o arquivo. |
| `Ctrl+X` | Sair do editor. |
| `Ctrl+K` | Recortar a linha ou seleção, conforme o contexto. |
| `Ctrl+6` | Iniciar/marcar seleção. |
| `Alt+\` | Ir para o início do arquivo. |
| `Alt+/` | Ir para o final do arquivo. |
| `Alt+U` | Desfazer uma edição. |

### Permissões e privilégios

A listagem detalhada exibiu permissões como `-rwxr-xr-x`:

- O primeiro `-` representa um arquivo comum.
- Os três blocos seguintes representam permissões de **dono**, **grupo** e **outros**.
- `r` = leitura, `w` = escrita, `x` = execução, `-` = permissão ausente.

O laboratório demonstrou duas situações distintas:

1. **Executar um script:** pode exigir `chmod +x` quando falta permissão de execução.
2. **Ler um arquivo protegido:** pode exigir autorização adicional; `sudo` executa o comando com privilégios elevados conforme as políticas do sistema.

## 8. Aplicações práticas em Segurança da Informação

| Competência | Exemplo de aplicação profissional |
|---|---|
| Inspeção prévia de scripts | Avaliar comandos de automação antes de executá-los em servidores. |
| Shell e expansão de variáveis | Criar rotinas repetíveis de coleta e triagem de evidências. |
| `grep` e leitura de logs | Identificar eventos relevantes em arquivos de registros. |
| Laços `for` | Automatizar verificações em conjuntos de arquivos. |
| Permissões Linux | Entender barreiras de acesso e aplicar o princípio do menor privilégio. |
| `bash -n` | Validar a sintaxe antes de um teste operacional. |
| Interpretação de erros | Distinguir problemas de sintaxe, caminhos, execução e autorização. |
| Numeração de linhas | Referenciar evidências de maneira verificável em análises e relatórios. |

**Limites do exercício:** o trabalho demonstrou fundamentos de automação e investigação local de logs. Não incluiu implementação de SIEM, monitoramento em produção, resposta automatizada a incidentes ou autenticação robusta.

## 9. Aprendizados e evolução

Ao final da sala, consegui:

- Entender a diferença entre **comando interativo** e **script salvo em arquivo**.
- Criar scripts que recebem entrada do usuário, utilizam variáveis e executam laços e condicionais.
- Entender a finalidade de `#!`, `$`, aspas, `./`, caminhos absolutos, curingas e operadores de comparação.
- Aplicar permissões de execução e reconhecer falhas de leitura por ausência de privilégio.
- Inspecionar e adaptar um script fornecido, validar sua sintaxe e interpretar seus resultados.
- Resolver problemas reais de edição no `nano`, sintaxe, caminhos e permissões sem simplesmente substituir a investigação por uma resposta pronta.
- Ler evidências diretamente em arquivos de log e verificar a origem da informação.

> **Principal conclusão:** o resultado mais valioso não foi a conclusão do desafio, mas a capacidade de diagnosticar *por que* um comando falhou, corrigir o problema e comprovar o funcionamento com uma nova execução.

## 10. Perguntas para revisão / entrevista

**Qual é a diferença entre Bash e shell?**  
*Shell* é a categoria de interface de comandos; Bash é uma implementação específica e uma linguagem de scripts.

**Por que `chmod +x` foi necessário?**  
Porque a execução direta `./arquivo.sh` requer permissão de execução. Uma alternativa para testar um arquivo legível é `bash arquivo.sh`, que solicita explicitamente ao Bash que interprete seu conteúdo; essa alternativa não substitui o entendimento das permissões.

**O que `grep -q` retorna?**  
Normalmente, status `0` quando há correspondência, `1` quando não há e um valor de erro quando ocorre problema. No `if`, o script utiliza o status de saída, não uma linha impressa.

**O que `bash -n` garante?**  
Somente que o Bash não detectou erros de sintaxe na análise; não prova permissões, ausência de falhas lógicas ou segurança do código.

**Por que `sudo` permitiu ler logs protegidos?**  
Porque o comando foi executado com privilégios elevados, sujeitos às regras de autorização da máquina. Em ambientes reais, deve-se limitar esse acesso ao necessário.

**`"$directory"/*.log` pesquisa subpastas?**  
Não. A expansão glob simples corresponde a entradas `.log` diretamente no diretório indicado, não recursivamente.

**Por que não devemos armazenar PINs em scripts?**  
Porque valores embutidos no código podem ser lidos por quem tiver acesso ao arquivo ou aos artefatos de distribuição. Uma aplicação real precisa de mecanismos adequados de autenticação e proteção de segredos.

---

### Registro e escopo

Este documento reflete os exercícios relatados e as saídas observadas durante a prática na sala *Linux Shells* do TryHackMe. Alguns trechos foram **reconstituídos ou sanitizados para publicação**; não devem ser confundidos com uma transcrição literal de todos os arquivos da máquina. Nenhuma flag ou credencial de laboratório foi incluída.

**Próxima etapa da jornada:** aplicar esses fundamentos a atividades de análise de logs, triagem de eventos e automações defensivas em ambientes autorizados.
