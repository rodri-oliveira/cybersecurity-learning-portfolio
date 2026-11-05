# Cryptography — TryHackMe | Relatório técnico

**Trilha:** Cryptography  
**Sala prática:** John the Ripper: The Basics  
**Data:** 05/11/2026  
**Ambiente:** TryHackMe AttackBox (Linux)  
**Objetivo profissional:** SOC Analyst Jr / Blue Team

> Este documento registra exercícios realizados em laboratório autorizado. Senhas recuperadas, passphrases, flags e hashes de desafios foram intencionalmente omitidos.

## Sumário

1. [Objetivo e fundamentos](#1-objetivo-e-fundamentos)
2. [Ferramentas](#2-ferramentas)
3. [Hashing Basics](#3-hashing-basics)
4. [John the Ripper — laboratório](#4-john-the-ripper--laboratório)
5. [Manual de comandos](#5-manual-de-comandos)
6. [Resultados e evidências](#6-resultados-e-evidências)
7. [Aplicações para SOC](#7-aplicações-para-soc)
8. [Limitações e próximos passos](#8-limitações-e-próximos-passos)
9. [Conclusão](#9-conclusão)

## 1. Objetivo e fundamentos

Estudar hashing, armazenamento seguro de senhas e auditoria offline de credenciais, com foco em interpretação de comandos e aplicação defensiva.

| Conceito | Resumo |
|---|---|
| Criptografia | Protege informações com algoritmos e chaves |
| Hash criptográfico | Produz um resumo; não se trata de criptografia reversível |
| Base64 | Codificação, **não** criptografia |
| Salt | Valor que dificulta ataques pré-computados e a correlação de senhas iguais |
| KDF | Derivação de chaves com custo computacional, como PBKDF2 |
| HMAC | Autenticidade/integridade com chave secreta e função hash |
| Ataque de dicionário | Teste de senhas candidatas a partir de uma lista |
| Ataque offline | Teste local, sem necessariamente gerar tentativas de login no serviço |

## 2. Ferramentas

- **John the Ripper (Jumbo):** auditoria de hashes.
- **rockyou.txt:** dicionário de senhas candidatas.
- **unshadow:** combina informações de `passwd` e `shadow`.
- **zip2john / rar2john / ssh2john.py:** preparam dados de arquivos protegidos.
- **unzip / unrar:** inspecionam arquivos compactados do laboratório.

Diretório utilizado:

```text
~/John-the-Ripper-The-Basics/
```

## 3. Hashing Basics

Os tópicos estudados anteriormente na trilha incluíram hashing versus criptografia, segurança de senhas, Hashcat, HMAC e Base64. Esta seção registra os **assuntos abordados**; não reconstrói saídas de terminal ausentes do histórico.

## 4. John the Ripper — laboratório

### Task 4 — Cracking Basic Hashes

Foram praticados testes com formatos MD5, SHA-1 e SHA-256:

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt ~/John-the-Ripper-The-Basics/Task04/hash1.txt
john --format=raw-sha1 --wordlist=/usr/share/wordlists/rockyou.txt ~/John-the-Ripper-The-Basics/Task04/hash2.txt
john --format=raw-sha256 --wordlist=/usr/share/wordlists/rockyou.txt ~/John-the-Ripper-The-Basics/Task04/hash3.txt
```

O quarto hash foi analisado como SHA-512 simples:

```bash
john --format=raw-sha512 --wordlist=/usr/share/wordlists/rockyou.txt ~/John-the-Ripper-The-Basics/Task04/hash4.txt
john --show --format=raw-sha512 ~/John-the-Ripper-The-Basics/Task04/hash4.txt
```

**Limitação:** o dicionário foi percorrido sem correspondência (`0 password hashes cracked, 1 left`). Uma tentativa com regras foi interrompida. **Não há evidência de recuperação do quarto hash**; portanto, não é apresentado como concluído.

### Task 5 — Windows Authentication Hashes

```bash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt ~/John-the-Ripper-The-Basics/Task05/ntlm.txt
```

**Conceitos:** NT Hash é a representação de senha usada em mecanismos de autenticação Windows; NTLM é uma família de protocolos de autenticação, não o mesmo conceito. O SAM armazena dados de contas locais.

### Task 6 — `/etc/shadow` e `unshadow`

No laboratório, `local_passwd` continha uma entrada para a conta `root` com UID e GID 0. `local_shadow` apresentava o prefixo `$6$`, correspondente a **SHA-512-crypt**, diferente de SHA-512 puro.

```bash
cat ~/John-the-Ripper-The-Basics/Task06/local_passwd
cat ~/John-the-Ripper-The-Basics/Task06/local_shadow
unshadow ~/John-the-Ripper-The-Basics/Task06/local_passwd ~/John-the-Ripper-The-Basics/Task06/local_shadow > ~/John-the-Ripper-The-Basics/Task06/unshadowed.txt
john --format=sha512crypt --wordlist=/usr/share/wordlists/rockyou.txt ~/John-the-Ripper-The-Basics/Task06/unshadowed.txt
john --show --format=sha512crypt ~/John-the-Ripper-The-Basics/Task06/unshadowed.txt
```

**Resultado:** uma senha de laboratório recuperada (`1g`); 5.000 iterações informadas pelo John. A senha foi omitida.

### Task 7 — Single Crack Mode

O modo `--single` utiliza atributos da conta para criar candidatos com transformações (*word mangling*). O exercício usou o usuário fictício `Joker` e um hash MD5.

```bash
# Exemplo de estrutura: substitua pelos dados autorizados do laboratório.
printf '%s\n' 'Joker:<HASH_DO_LABORATORIO>' > hash07-single.txt
john --single --format=raw-md5 hash07-single.txt
```

**Resultado:** uma senha recuperada (`1g`) com substituição previsível de caracteres. **Aprendizado:** *leetspeak* não transforma uma senha previsível em senha forte.

### Task 8 — Custom Rules

Foram estudadas regras de transformação de palavras:

| Regra | Função |
|---|---|
| `c` | Capitaliza uma palavra |
| `Az` | Acrescenta caracteres ao final |
| `A0` | Acrescenta caracteres ao início |
| `[A-Z]` | Letras maiúsculas |
| `[0-9]` | Dígitos |

Exemplo estudado:

```text
Az"[A-Z]"
```

**Aprendizado:** regras personalizadas exploram a previsibilidade dos padrões de senhas. A sintaxe do comando de execução deve ser conferida na versão instalada do John.

### Task 9 — ZIP protegido

```bash
zip2john ~/John-the-Ripper-The-Basics/Task09/secure.zip > ~/John-the-Ripper-The-Basics/Task09/zip_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt ~/John-the-Ripper-The-Basics/Task09/zip_hash.txt
john --show ~/John-the-Ripper-The-Basics/Task09/zip_hash.txt
```

O `zip2john` identificou o conteúdo interno `zippy/flag.txt` e criptografia PKZIP tradicional. **Resultado:** `1 password hash cracked, 0 left`; senha e flag omitidas.

### Task 10 — RAR protegido

```bash
rar2john ~/John-the-Ripper-The-Basics/Task10/secure.rar > ~/John-the-Ripper-The-Basics/Task10/rar_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt ~/John-the-Ripper-The-Basics/Task10/rar_hash.txt
unrar l ~/John-the-Ripper-The-Basics/Task10/secure.rar
unrar p ~/John-the-Ripper-The-Basics/Task10/secure.rar flag.txt
```

O hash continha o identificador `$rar5$`. O John identificou **RAR5 / PBKDF2-SHA256** e **32.768 iterações**. **Resultado:** uma senha recuperada (`1g`) e `flag.txt` identificado; conteúdo omitido.

### Task 11 — Chave SSH protegida

O arquivo `id_rsa` tinha permissões `644`, inadequadas para uma chave privada real, por permitir leitura por grupo e outros usuários.

```bash
ls -l ~/John-the-Ripper-The-Basics/Task11/
python3 /opt/john/ssh2john.py ~/John-the-Ripper-The-Basics/Task11/id_rsa > ~/John-the-Ripper-The-Basics/Task11/id_rsa_hash.txt
ls -lh ~/John-the-Ripper-The-Basics/Task11/id_rsa_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt ~/John-the-Ripper-The-Basics/Task11/id_rsa_hash.txt
```

**Resultado:** uma passphrase recuperada (`1g`); valor omitido. O arquivo preparado tinha cerca de 2,5 KB.

Medida defensiva para uma chave privada legítima:

```bash
chmod 600 ~/.ssh/id_rsa
```

Uma **passphrase SSH** não é necessariamente a senha de login do usuário no Linux.

## 5. Manual de comandos

| Comando ou sintaxe | Descrição |
|---|---|
| `ls -l` | Lista arquivos com permissões, proprietário, tamanho e data |
| `ls -lh` | Exibe tamanhos em unidades legíveis |
| `cat ARQUIVO` | Mostra o conteúdo de um arquivo |
| `john` | Executa o John the Ripper |
| `--format=TIPO` | Define formato do hash |
| `--wordlist=CAMINHO` | Define dicionário de candidatos |
| `--single` | Ativa o Single Crack Mode |
| `--show` | Consulta senhas recuperadas previamente |
| `--rules` | Ativa regras de transformação |
| `unshadow` | Combina dados de `passwd` e `shadow` |
| `zip2john` / `rar2john` | Preparam hashes dos arquivos compactados |
| `ssh2john.py` | Prepara dados de chave SSH protegida |
| `unrar l` | Lista arquivos do RAR |
| `unrar p` | Mostra arquivo interno sem gravá-lo em disco |
| `>` | Redireciona saída, sobrescrevendo o arquivo de destino |
| `~` | Diretório pessoal |
| `Ctrl+C` | Interrompe o programa em primeiro plano |
| `chmod 600` | Concede leitura/escrita somente ao proprietário |

**Interpretando status:** `Loaded 1 password hash` = hash reconhecido; `1g` = senha recuperada; `0g` = nenhuma senha recuperada até o ponto observado; `DONE` = execução encerrada.

## 6. Resultados e evidências

| Exercício | Resultado observado |
|---|---|
| Task 4 | MD5, SHA-1 e SHA-256 praticados; quarto hash SHA-512 não recuperado |
| Task 5 | NT Hash testado no laboratório |
| Task 6 | Senha da conta didática recuperada (`1g`) |
| Task 7 | Senha recuperada em Single Crack Mode (`1g`) |
| Task 8 | Sintaxe e conceitos de regras personalizadas estudados |
| Task 9 | ZIP: senha recuperada e arquivo interno identificado |
| Task 10 | RAR5: senha recuperada e arquivo interno identificado |
| Task 11 | Chave SSH: passphrase recuperada (`1g`) |

O relatório documenta resultados observados no terminal; não inventa capturas de tela ou comprovações. A conclusão da trilha foi informada pelo estudante e não corresponde a uma verificação independente do progresso na plataforma.

## 7. Aplicações para SOC

1. **Senhas fortes:** evitar palavras comuns, nomes e substituições triviais.
2. **Controle de acesso:** restringir acesso a hashes, cópias de credenciais e chaves privadas.
3. **Gestão de credenciais:** quando há exposição, considerar rotação/revogação e análise do impacto.
4. **Monitoramento:** ataques offline podem ocorrer sem gerar falhas de autenticação no servidor.
5. **Documentação técnica:** registrar comandos, resultados, limitações e recomendações sem publicar segredos.

## 8. Limitações e próximos passos

**Dificuldade documentada:** a tentativa de recuperação do quarto hash SHA-512 na Task 4 terminou sem correspondência no dicionário utilizado. Um ataque com regras foi interrompido. O exercício não deve ser apresentado como resolvido sem uma evidência nova.

Próximos estudos recomendados: comparar SHA-512 puro, SHA-512-crypt, PBKDF2, bcrypt, scrypt e Argon2; praticar regras personalizadas em laboratório; estudar rotação de chaves SSH; correlacionar alertas de autenticação e exposição de credenciais em SIEM/EDR.

## 9. Conclusão

A experiência conectou conceitos de criptografia e hashing à auditoria de senhas em ambientes Windows e Linux, arquivos ZIP/RAR e chaves SSH. Os resultados reforçaram que **algoritmos adequados não compensam credenciais previsíveis nem arquivos sensíveis expostos**.

O aprendizado é relevante à atuação de **SOC Analyst Jr / Blue Team** por desenvolver raciocínio analítico, interpretação de saídas, entendimento de riscos de credenciais e documentação técnica reproduzível.

---

*Uso educacional e autorizado. Sem divulgação de senhas, passphrases, flags ou chaves privadas.*
