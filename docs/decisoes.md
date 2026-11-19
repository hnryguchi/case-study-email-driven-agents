# Decisões (estilo ADR)

**Português** · [English](decisions.md)

Registros curtos das decisões que moldaram os agentes. Cada uma se baseia no código em funcionamento; o código em si não é publicado.

## 1. Motor de regras primeiro, LLM como segunda opinião

**Contexto.** A revisão de registros de monitoramento tem dois tipos de verificação: as objetivas (horários contra o log de operações, aritmética das interrupções, estados da fonte) e as que exigem julgamento (essa sequência de eventos é biologicamente plausível, a narrativa é coerente com os dados). Os revisores independentes anteriores cobriam apenas o primeiro tipo.

**Decisão.** Manter o motor determinístico como primeira camada e enviar seus achados, com um dossiê por registro, a um LLM que precisa confirmar ou descartar cada achado com uma justificativa e uma fonte, e que pode acrescentar seus próprios itens classificados como Erro, Validação humana ou Sugestão.

**Consequências.** As verificações objetivas continuam reprodutíveis e baratas. A contribuição do modelo fica visível e auditável item a item. Se o modelo estiver indisponível, os achados das regras ainda são entregues. O modelo pode errar, então sua saída nunca é a palavra final (veja a 2).

## 2. Os agentes criam rascunhos; uma pessoa envia

**Contexto.** Os resultados da revisão vão para revisores seniores e, indiretamente, para as equipes dos navios e para a base de dados do órgão regulador. Uma mensagem errada custa caro e é difícil de desfazer.

**Decisão.** O redator escreve em uma cópia da planilha de controle de revisão e cria um rascunho no Outlook. O código não tem caminho de envio.

**Consequências.** A pior falha automatizada é um rascunho ruim que uma pessoa descarta. O preço é que uma pessoa precisa abrir e enviar cada resposta, o que é aceitável porque essa revisão é justamente o objetivo.

## 3. Copiar as entradas, nunca alterar os originais

**Contexto.** As planilhas são registros regulatórios e pastas de trabalho com macros. Abri-las no lugar arrisca bloqueios, recálculos ou salvamentos acidentais.

**Decisão.** O revisor copia cada entrada para um diretório temporário antes da leitura. O watcher apenas lê e-mails. O redator escreve somente em uma cópia renomeada.

**Consequências.** Reexecutar qualquer etapa é seguro. Os anexos salvos e as cópias se acumulam em disco em pastas por mensagem, então a limpeza é uma tarefa manual.

## 4. Despacho idempotente com arquivos de lock e conjuntos de itens vistos

**Contexto.** O watcher roda a cada cinco minutos, o Outlook pode ressincronizar itens antigos, uma revisão pode levar mais que o intervalo de consulta, e o mesmo reporte pode chegar duas vezes.

**Decisão.** Deduplicar e-mails pelo `EntryID` (limitado aos últimos 3.000 itens); deduplicar revisões por um hash SHA-256 dos arquivos de entrada; um arquivo de lock por tipo de agente, com limite de validade; um flag de "repetir ao terminar" em vez de execuções simultâneas; publicação apenas quando um reporte é mais recente que o publicado.

**Consequências.** Nenhuma revisão, rascunho ou publicação duplicados, e nenhum acúmulo de processos simultâneos. O estado fica em pequenos arquivos JSON em uma única máquina, o que amarra o sistema àquela estação de trabalho.

## 5. Dois caminhos de acesso ao LLM, degradação explícita

**Contexto.** O revisor roda na minha estação de trabalho e nas máquinas de colegas, por meio de um instalador. Nem toda máquina tem uma chave de API, e chaves não devem ficar embutidas em um executável distribuído.

**Decisão.** Usar uma chave de API se houver uma configurada localmente; caso contrário, chamar o Claude Code CLI logado em modo headless, sem ferramentas, com um schema JSON de saída e um diretório de trabalho vazio; caso contrário, rodar apenas as regras. O relatório informa qual modo foi usado.

**Consequências.** Nenhum segredo compartilhado é distribuído no instalador, e o revisor continua entregando valor sem nenhum acesso a modelo. Dois caminhos de código precisam ser mantidos em sincronia, o que a normalização da saída ajuda a conter.
