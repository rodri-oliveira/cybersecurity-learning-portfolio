# Padrão para evidências de laboratório (SOC L1)

**Objetivo:** demonstrar capacidade de observar, investigar e comunicar resultados sem expor dados confidenciais.

## Estrutura recomendada

| Campo | Como preencher |
| --- | --- |
| Contexto | Qual foi a pergunta técnica investigada? |
| Escopo | Qual ambiente autorizado foi utilizado? |
| Fonte | Sistema, log, ferramenta ou comando responsável pela observação |
| Momento | Data da análise, sem expor dados pessoais |
| Procedimento | Passos reproduzíveis e explicação dos comandos |
| Evidência | Trecho sanitizado da saída ou captura |
| Interpretação | Conclusões estritamente sustentadas pelas observações |
| Limitações | O que ainda não pode ser concluído? |
| Próximo passo | Qual verificação seria necessária para aumentar a confiança? |

## Exemplo **fictício** de relato

**Pergunta:** há indícios de falhas repetidas de autenticação no ambiente de teste?

**Observação ilustrativa:** um conjunto sintético de logs apresenta múltiplas tentativas de login rejeitadas, seguido de uma autenticação bem-sucedida.

**Interpretação:** o padrão merece investigação, mas, sozinho, não confirma uma invasão. É preciso verificar origem, horário, comportamento habitual e outras fontes correlacionadas.

**Limitação:** exemplo didático, **não é evidência de um incidente real** e não deve ser apresentado como resultado executado.

## Não publicar

Credenciais ou tokens; números de série; dados pessoais; IPs ou endpoints privados; dados corporativos; registros brutos de aparelhos; flags ou gabaritos de laboratórios.

Ver também [política de segurança](../SECURITY.md). Antes de enviar qualquer material, revise o diff do Git.
