# Active Directory Basics (TryHackMe)

Trilha: Cyber Security 101 · Módulo: Windows and AD Fundamentals

Resumo do que aprendi e pratiquei nesta sala. Não inclui flags nem respostas das perguntas.

## 1. Conceitos

### O que é o Active Directory (AD)
Serviço da Microsoft que centraliza usuários, computadores e permissões de uma rede corporativa. Em vez de ter uma conta em cada computador, a empresa tem um **domínio** e as credenciais ficam guardadas em um único lugar.

### Domain Controller (DC)
O servidor que executa os serviços do AD. Quando alguém faz login em qualquer máquina do domínio, o DC confere o usuário e a senha. Por concentrar todas as credenciais, é o alvo mais valioso em um ataque.

### Objetos do AD
- **Usuários:** pessoas (e contas de serviço).
- **Contas de máquina:** cada computador do domínio tem uma. O nome é o nome do computador seguido de `$` (exemplo: `PC-MARK$`).
- **Grupos:** conjuntos de usuários com as mesmas permissões. O grupo com poder máximo no domínio é o **Domain Admins**.

### Organizational Units (OUs)
"Pastas" dentro do AD para agrupar usuários, computadores e grupos (por departamento, por exemplo). Servem para aplicar **políticas** a todo o grupo de uma vez.

### Group Policy Objects (GPOs)
Conjuntos de regras (senhas, bloqueios, configurações) aplicados a usuários e computadores de uma OU. Ficam armazenadas em um compartilhamento de rede no DC, o **SYSVOL**, de onde as máquinas do domínio as leem.

### Autenticação: Kerberos e NetNTLM
- **Kerberos:** padrão nas versões atuais do Windows. O usuário recebe um ticket inicial (TGT) e, com ele, pede tickets de serviço (TGS) para cada recurso que quiser usar.
- **NetNTLM:** protocolo antigo, mantido por compatibilidade. Usa desafio e resposta, então a senha em si não trafega pela rede.

### Estrutura entre domínios
- **Tree:** domínios que compartilham o mesmo namespace (exemplo: `thm.local` e `uk.thm.local`).
- **Forest:** conjunto de árvores.
- **Trust:** relação de confiança configurada entre domínios para que usuários de um acessem recursos do outro.

## 2. Prática no laboratório

### Acesso ao laboratório
1. Iniciar a **Lab machine** da sala e anotar o IP dela (em *Active machines information*).
2. No AttackBox (Linux), abrir o **Remmina** e conectar por RDP:
   - Server: IP da Lab machine
   - Username: `Administrator` · Domain: `THM`
   - Password: fornecida pela sala
3. Aceitar o certificado autoassinado.

Atenção: o IP do AttackBox e o IP da Lab machine são diferentes. Usar o do AttackBox causa "Failed to connect to the RDP server".

### Delegação de controle
Dar a um usuário um poder específico sobre uma OU, sem torná-lo administrador (princípio do menor privilégio).

1. No *Active Directory Users and Computers*, abrir `thm.local > THM`.
2. Botão direito na OU **Sales** > **Delegate Control**.
3. Adicionar o usuário `phillip` (suporte de TI) com **Check Names**.
4. Marcar a tarefa **Reset user passwords and force password change at next logon**.

### Redefinir a senha de outro usuário (como Phillip, via PowerShell)
```powershell
Set-ADAccountPassword sophie -Reset -NewPassword (Read-Host -AsSecureString -Prompt 'New Password') -Verbose
Set-ADUser -ChangePasswordAtLogon $true -Identity sophie -Verbose
```
O primeiro comando pede a nova senha (não aparece na tela). O segundo obriga o usuário a trocá-la no próximo login.

Resultado: o Phillip, que não é administrador, conseguiu redefinir a senha da Sophie apenas por causa da delegação.

### Organização de computadores em OUs
1. Os computadores novos ficam soltos na pasta **Computers**.
2. Criar as OUs **Workstations** (PCs e notebooks) e **Servers** (servidores) com botão direito em `thm.local` > **New > Organizational Unit**.
3. Arrastar cada computador para a OU correspondente.

Boa prática: separar servidores e estações de trabalho, porque cada grupo precisa de políticas de segurança diferentes.

## 3. Problemas que encontrei e como resolvi

| Problema | Causa | Solução |
|---|---|---|
| `Failed to connect to the RDP server` | Usei o IP do AttackBox em vez do IP da Lab machine | Usar o IP da Lab machine |
| UAC pedindo senha de administrador ao abrir o PowerShell | PowerShell aberto com elevação | Clicar em **No** e abrir o PowerShell normal |
| `The password does not meet the length, complexity, or history requirement` | Senha fraca para a política do domínio | Usar senha com maiúscula, minúscula, número e símbolo |
| `Change user password before connecting` no Remmina | `ChangePasswordAtLogon $true` bloqueia o RDP, que não mostra a tela de troca | Rodar o comando de novo com `$false` |

## 4. O que aprendi
- Delegação permite dar poder específico sem dar acesso de administrador.
- A política de senha do domínio recusa senhas fracas.
- Contas de máquina terminam com `$`.
- Servidores e estações de trabalho devem ficar em OUs separadas.
- O AD é o alvo central em ataques a redes Windows, por isso o menor privilégio importa.
