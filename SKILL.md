---
name: sync-granola-tana
description: Sincroniza reunioes do Granola para Daily Pages no Tana
disable-model-invocation: true
---

# Skill: Sync Granola -> Tana

Sincroniza reunioes dos ultimos 7 dias do Granola para os Daily Pages correspondentes no Tana, criando entradas formatadas com horario, titulo e resumo.

## Principios

1. **Nunca apagar conteudo do Tana** - Apenas adicionar, nunca remover
2. **Evitar duplicatas** - Verificar antes de criar
3. **Confirmar antes de agir** - Mostrar plano ao usuario antes de executar
4. **Fuso horario Brasil** - Todos os horarios em America/Sao_Paulo (BRT, UTC-3)

---

## Workflow

### Fase 1: Verificar Conectividade

Antes de qualquer coisa, verificar se os MCPs estao acessiveis:

1. **Granola MCP**: Chamar `list_meetings` com `time_range: "last_30_days"` (usamos 30 dias e filtramos para 7)
   - Se falhar: informar usuario que o MCP do Granola nao esta acessivel
2. **Tana MCP**: O MCP do Tana roda como servidor HTTP local na porta 8262. Chamar via curl:
   - Primeiro: `list_workspaces` para obter o workspaceId principal
   - Depois: `get_or_create_calendar_node` com `workspaceId`, `date`, `granularity: "day"`, `createIfMissing: true`
   - Headers necessarios: `Accept: application/json, text/event-stream` e `Authorization: Bearer {token}`
   - Se falhar: informar usuario que o MCP do Tana nao esta respondendo na porta 8262
   - **NOTA**: As tools do Tana MCP podem nao estar disponiveis como MCP tools nativos do Claude Code. Nesse caso, usar chamadas HTTP diretas via curl com o endpoint `http://127.0.0.1:8262/mcp` e o protocolo JSON-RPC.

Se ambos funcionarem, prosseguir.

---

### Fase 2: Coletar Reunioes do Granola

1. Chamar `list_meetings` com `time_range: "last_30_days"`
2. Filtrar apenas reunioes dos ultimos 7 dias (calcular data de corte: hoje - 7 dias)
3. Para cada reuniao, extrair:
   - `id`: ID da reuniao
   - `title`: Titulo
   - `date`: Data e horario (converter para BRT se necessario)
   - `participants`: Lista de participantes

4. Para cada reuniao, chamar `get_meetings` com o ID para obter:
   - `summary`: Resumo/notas completas
   - `private_notes`: Notas privadas (se houver)

**Otimizacao**: Fazer chamadas `get_meetings` em lotes de ate 10 IDs por chamada.

5. Montar lista estruturada:

```
reunioes = [
  {
    id: "abc123",
    titulo: "1:1 com Fulano",
    data: "2026-02-19",
    horario: "11:00",
    participantes: ["Fulano", "Ciclano"],
    resumo: "Texto do resumo...",
    notas_privadas: "Texto das notas..."
  },
  ...
]
```

---

### Fase 3: Verificar o que ja existe no Tana

**Importante**: A API requer `workspaceId` e `granularity`. Primeiro chamar `list_workspaces` para obter o workspace principal do usuario, e usar `granularity: "day"`.

Para cada data unica nas reunioes:

1. Chamar `get_or_create_calendar_node` com `workspaceId`, `date` (formato ISO: "YYYY-MM-DD"), `granularity: "day"`, e `createIfMissing: true`
2. Obter o `nodeId` do Daily Page
3. Chamar `get_children` com o `nodeId` do Daily Page para listar entradas existentes
4. Para cada filho, verificar se ja foi sincronizado do Granola

**Criterio de match - PRIORIDADE (reuniao ja existe)**:

1. **Match por link Granola (mais confiavel)**: Chamar `read_node` com `maxDepth: 2` para cada filho do Daily Page. Se o conteudo contiver `notes.granola.ai/t/{meeting-id}`, e um match exato. O Granola meeting ID esta no link.

2. **Match por titulo/participantes (fallback)**: Se nao encontrar link Granola:
   - O titulo do node filho contem o nome de um participante da reuniao Granola, OU
   - O titulo do node filho contem palavras-chave do titulo da reuniao (case-insensitive, minimo 2 palavras)

3. **Match por horario (fraco, pedir confirmacao)**: Mesmo horario mas titulo diferente -> perguntar ao usuario

**NOTA**: Os horarios no Tana podem diferir do Granola! O usuario pode ter criado a entrada manualmente com horario diferente. Nao confiar apenas no horario para matching.

Classificar cada reuniao como:
- **JA SINCRONIZADA**: Link Granola encontrado no node -> ignorar
- **NOVA**: Nenhum match encontrado -> sera criada
- **POSSIVEL DUPLICATA**: Match parcial -> perguntar ao usuario

---

### Fase 4: Apresentar Plano ao Usuario

Antes de fazer qualquer alteracao, mostrar relatorio:

```
## Plano de Sincronizacao Granola -> Tana

**Periodo**: [data inicio] a [data fim]
**Reunioes encontradas no Granola**: X

### Novas (serao criadas no Tana):
1. 19/02 11:00 - 1:1 com Fulano
2. 19/02 14:00 - Planning Sprint 42

### Existentes (receberao notas do Granola):
1. 18/02 10:00 - Daily Standup (ja existe no Tana)

### Ignoradas:
(nenhuma)

Confirma a sincronizacao? (S/N)
```

Usar `AskUserQuestion` para confirmar. Aguardar resposta antes de prosseguir.

---

### Fase 5: Executar Sincronizacao

#### Para reunioes NOVAS:

Usar `import_tana_paste` para criar a entrada no Daily Page.

**Parametros da API**: `workspaceId`, `parentNodeId` (nodeId do Daily Page), `content` (Tana Paste string)

Formato Tana Paste:

```
- HH:MM [Titulo da Reuniao] #meeting
  - **Date**: [Dia da semana abreviado], [Mes] [Dia]
  - **Attendees**: [Nome participante 1], [Nome participante 2]
  - Summary
    - [Secao 1 do resumo]
      - [Detalhes como nodes filhos]
    - [Secao 2 do resumo]
      - [Detalhes como nodes filhos]
    - Chat with meeting transcript: https://notes.granola.ai/t/[granola-meeting-id]
```

**Regras de formatacao do resumo**:
- Usar tag `#meeting` no node raiz da reuniao
- Incluir campo **Date** e **Attendees** como filhos diretos
- Cada secao do resumo (separada por headers ###) vira um node filho sob "Summary"
- Manter a estrutura de bullet points do resumo original
- Preservar formatting basico (bold, italico via ** **)
- Nao incluir timestamps internos do Granola (ex: [00:52:30])
- **SEMPRE incluir link Granola como ultimo filho de Summary**: `Chat with meeting transcript: https://notes.granola.ai/t/{meeting-id}`
  - Este link e essencial para o matching de duplicatas em futuras sincronizacoes!

O `parentNodeId` de destino e o Daily Page obtido na Fase 3.

Ver `references/tana-paste-format.md` para detalhes da sintaxe.

#### Para reunioes EXISTENTES:

1. Localizar o node existente no Tana (ja identificado na Fase 3)
2. Chamar `read_node` para verificar se ja tem "!Summary Granola" como filho
3. Se NAO tem "!Summary Granola":
   - Usar `import_tana_paste` com `nodeId` do node existente para adicionar:
     ```
     - !Summary Granola
       - [Conteudo do resumo]
     ```
4. Se JA tem "!Summary Granola":
   - Informar usuario que ja existe resumo e perguntar se deseja substituir
   - Se sim: usar `trash_node` no node "!Summary Granola" antigo e criar novo
   - Se nao: pular esta reuniao

---

### Fase 6: Relatorio Final

Apos execucao, apresentar relatorio:

```
## Sincronizacao Concluida

**Criadas**: X reunioes
**Atualizadas**: Y reunioes
**Ignoradas**: Z reunioes

### Detalhes:
- [check] 19/02 11:00 - 1:1 com Fulano (criada)
- [check] 19/02 14:00 - Planning Sprint 42 (criada)
- [check] 18/02 10:00 - Daily Standup (notas adicionadas)
```

---

## Tratamento de Erros

### Granola MCP indisponivel
Informar: "O MCP do Granola nao esta respondendo. Verifique se o Granola esta aberto e o servidor MCP esta rodando."

### Tana MCP indisponivel
Informar: "O MCP do Tana nao esta respondendo. Verifique se o servidor Tana MCP local esta rodando na porta 8262."

### Daily Page nao encontrado
Criar automaticamente usando `get_or_create_calendar_node` com `createIfMissing: true`.

### Erro ao importar para Tana
Registrar o erro, continuar com as demais reunioes, e reportar no relatorio final quais falharam.

### Reuniao sem resumo no Granola
Criar a entrada no Tana apenas com titulo e horario, sem node "!Summary Granola". Informar no relatorio.

---

## Conversao de Horarios

**CRITICO**: A API do Granola retorna horarios em **UTC**, nao em horario local!

O Granola retorna datas no formato: `"Feb 19, 2026 2:00 PM"` (UTC)

**Sempre subtrair 3 horas** para converter UTC -> America/Sao_Paulo (BRT):

Exemplos:
- `11:00 AM` UTC -> `08:00` BRT
- `12:00 PM` UTC -> `09:00` BRT
- `2:00 PM` UTC -> `11:00` BRT
- `9:29 PM` UTC -> `18:29` BRT

Converter para:
- **Data ISO**: `2026-02-19` (para `get_or_create_calendar_node`) - a data pode mudar se o horario UTC for antes das 03:00 AM!
- **Horario display**: `HH:MM` em formato 24h BRT (para o titulo no Tana)

Passos de conversao:
1. Parse `"Feb 19, 2026 2:00 PM"` -> hora 14:00 UTC
2. Subtrair 3h -> 11:00 BRT
3. Se hora resultante < 0, subtrair 1 dia da data e somar 24h ao horario
4. Formatar como `11:00` (24h, sem AM/PM)

---

## Notas Importantes

- **NUNCA apague conteudo pre-existente no Tana** a menos que o usuario confirme explicitamente
- **Notas privadas do Granola** (`private_notes`): NAO incluir por padrao. Incluir apenas o `summary`.
- **Reunioes muito longas**: Se o resumo tiver mais de 5000 caracteres, avisar o usuario e perguntar se deseja importar completo ou resumido
- **Execucao idempotente**: Rodar o skill multiplas vezes nao deve criar duplicatas
