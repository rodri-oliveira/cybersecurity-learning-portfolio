# SOC L1 — Alert Triage & Investigation | Case Study 001

> Laboratório educacional simulado — TryHackMe, SOC Level 1 / Security Analyst Intro. Não representa experiência profissional nem investigação de infraestrutura real.

## 1. Escopo e objetivo

A atividade reproduz a rotina de um analista de segurança em um turno de trabalho: passagem de turno, consulta de inteligência de ameaças, priorização de alertas, investigação inicial, escalonamento e documentação.

O objetivo foi interpretar as evidências apresentadas pela simulação e justificar as decisões de triagem. Não houve coleta independente de logs brutos, execução de comandos ou consulta externa de reputação.

## 2. Resumo operacional

| Horário simulado | Atividade | Resultado |
|---|---|---|
| 08:00 | Passagem de turno e consulta de notícias | Priorização conforme risco |
| 10:00 | Investigação de acesso suspeito | Escalonamento para revisão |
| 12:00 | Investigação de falhas de autenticação | Falso alarme no contexto do laboratório |
| 14:00 | Encerramento e passagem de turno | Registro das decisões |

O painel final do exercício informa 12 alertas triados, uma ameaça encontrada e um falso alarme descartado. Esses são indicadores da simulação; apenas os dois casos abaixo foram detalhados nas evidências disponibilizadas.

## 3. Priorização inicial

| Alerta | Prioridade atribuída pela simulação |
|---|---|
| Unusual login location | Alta |
| New device registered | Média |
| Password changed | Média |
| Failed VPN login ×3 | Baixa |
| Antivirus definition update | Baixa |

A fila foi organizada por importância, não por ordem de chegada ou facilidade de encerramento. Em produção, a prioridade também dependeria da criticidade do ativo, do impacto potencial e do contexto do usuário.

A consulta de notícias destacou vulnerabilidades críticas em AD FS e SharePoint. A orientação do exercício era priorizar alertas relacionados e verificar o estado das correções de segurança em serviços expostos à internet. Não foram fornecidos identificadores CVE nem evidências de exploração na organização simulada.

## 4. Caso 001-A — Suspicious Login

### Contexto

Alerta de acesso originado de uma localização incomum, Frankfurt, onde a organização simulada não possui escritório.

### Evidências apresentadas

| Campo | Valor |
|---|---|
| IP de origem observado | `185.220.101.47` |
| Localização atribuída ao IP | Frankfurt, Alemanha |
| Prioridade inicial | Alta |
| Pontuação de ameaça | 92/100 |
| Classificação de reputação | Malicious |
| Tipo de rede | Tor exit node |
| Denúncias anteriores | 4: duas de brute force, uma de port scan e uma de credential stuffing |

A reputação acima foi fornecida pelo exercício, não verificada em fontes externas. Ela não constitui uma avaliação atual do endereço IP.

### Análise e decisão

A localização incomum motivou a investigação, mas não foi tratada isoladamente como prova de ataque. A presença de um nó de saída Tor indica que o endereço observado pode não corresponder à localização real da origem. O uso de Tor, por si só, não demonstra intenção maliciosa.

Foram selecionadas as três evidências solicitadas pela simulação para justificar o escalonamento: nó de saída Tor conhecido, pontuação de ameaça elevada e classificação de reputação maliciosa. A pontuação e a classificação são resultados da mesma consulta, não necessariamente fontes independentes.

**Disposição:** alerta suspeito escalonado para revisão; prioridade alta. A simulação contabilizou uma ameaça encontrada, mas as evidências disponíveis não confirmam comprometimento efetivo da conta.

### Recomendações para investigação em produção (não executadas)

- Consultar logs de autenticação para identificar conta, horário, resultado do acesso, dispositivo e eventos relacionados.
- Verificar histórico de acessos, uso autorizado de VPN/Tor e confirmação do usuário por canal confiável.
- Examinar eventos de autenticação multifator e sessões posteriores.
- Se houver evidência de comprometimento, aplicar medidas de contenção conforme o procedimento e as autorizações da organização.

## 5. Caso 001-B — Authentication False Positive

### Contexto e evidências

| Campo | Valor |
|---|---|
| Ativo | `APP-SERVER-04` |
| Evento | 8 falhas de autenticação em 3 minutos |
| Chamado técnico | `IT-4471` |
| Mudança documentada | Rotação de senha de uma conta de automação no dia anterior |
| Causa apresentada | Scripts continuaram utilizando a senha antiga |

### Linha do tempo

1. Dia anterior: equipe de TI altera a senha da conta de automação.
2. Madrugada: scripts continuam tentando autenticar com a credencial antiga.
3. Dia do turno: monitoramento gera alerta de oito falhas em três minutos.

### Análise e decisão

A concentração de falhas poderia ser compatível com uma tentativa de força bruta. Entretanto, o chamado técnico fornece uma explicação operacional coerente para o evento. O exercício confirma que o alerta foi um falso alarme relacionado à configuração incorreta de um script.

**Disposição:** falso positivo quanto à hipótese de ataque, conforme a simulação. As falhas de autenticação são eventos reais dentro do cenário; a interpretação maliciosa é que foi descartada.

### Recomendações para produção (não executadas)

- Correlacionar conta, origem, horários e ativo dos eventos com o chamado antes de encerrar o alerta.
- Solicitar à equipe responsável a atualização segura da credencial utilizada pela automação.
- Verificar se as falhas cessaram após a correção e reabrir a investigação caso persistam sem explicação.
- Não criar exclusões amplas de detecção apenas para evitar alertas recorrentes.

## 6. Passagem de turno

- Acesso suspeito: encaminhado para revisão; conclusão sobre comprometimento pendente.
- Falhas de autenticação: classificadas como falso alarme no exercício.
- Demais tarefas: fila indicada como sem pendências pela simulação.

## 7. Competências exercitadas e limites

- Alert triage: avaliação inicial e classificação de alertas.
- Alert prioritization: organização da fila conforme risco.
- Threat intelligence: interpretação de informações sobre ameaças.
- IP reputation analysis: interpretação de reputação de endereços IP.
- Event correlation: relação entre alerta e mudança operacional documentada.
- False positive analysis: distinção entre atividade suspeita e causa legítima.
- Escalation and shift handover: encaminhamento e transferência de contexto.

O exercício foi guiado e não demonstrou operação independente de SIEM, análise de logs brutos ou resposta efetiva em produção. Essas competências deverão ser aprofundadas em laboratórios posteriores.

## Referência

TryHackMe — Security Analyst Intro, trilha SOC Level 1. Evidências: telas do diário interativo e painel de encerramento apresentados durante o exercício. Flag de conclusão omitida deliberadamente.
