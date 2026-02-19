# Tana Paste Format for Meeting Import

Reference for formatting Granola meetings into Tana Paste syntax.

---

## Basic Structure

Tana Paste uses indentation (2 spaces) to represent hierarchy. Each line starting with `- ` is a node.

```
- Parent node
  - Child node
    - Grandchild node
```

---

## Meeting Entry Format

### New Meeting (full entry)

```
- 11:00 1:1 com Fulano
  - Participantes: Fulano, Ciclano
  - !Summary Granola
    - [resumo content as child nodes]
```

### Adding Summary to Existing Meeting

When the meeting node already exists in Tana, import into it:

```
- !Summary Granola
  - [resumo content as child nodes]
```

Use `nodeId` parameter of `import_tana_paste` to target the existing meeting node.

---

## Formatting Rules for Summary Content

### Headers become bold parent nodes

Granola summary often has `### Section Title` headers. Convert to:

```
- **Section Title**
  - Content under this section
```

### Bullet points stay as nodes

```
- **Abertura / Check-in**
  - Viagem intensa para desmontar apartamento
  - Carnaval na praia: saida as 3h50
  - Trabalho na ferramenta durante o carnaval
```

### Nested bullets preserve hierarchy

```
- **Aprendizados-chave**
  - O metodo funciona como medida de protecao
  - Diferenca entre mentalidade executiva vs empreendedora
```

### Key-value pairs

```
- **Humor geral**: 8/10
- **Energia**: 8/10
```

### Bold text within content

Keep `**text**` as-is, Tana renders it.

---

## What to Strip

1. **Timestamps**: Remove `[00:52:30]` style timestamps
2. **Markdown links**: Convert `[text](url)` to just `text (url)`
3. **Horizontal rules**: Remove `---` separators
4. **Empty lines**: Skip empty content

---

## Import API Usage

### Creating new entry in Daily Page

```
import_tana_paste(
  paste: "- 11:00 1:1 com Fulano\n  - Participantes: Fulano, Ciclano\n  - !Summary Granola\n    - **Resumo da reuniao**\n      - Ponto 1\n      - Ponto 2",
  nodeId: "[daily-page-node-id]"
)
```

### Adding summary to existing meeting node

```
import_tana_paste(
  paste: "- !Summary Granola\n  - **Resumo da reuniao**\n    - Ponto 1\n    - Ponto 2",
  nodeId: "[existing-meeting-node-id]"
)
```

---

## Character Limits

- Tana Paste has no hard limit, but very large imports may timeout
- If summary > 5000 chars, consider splitting into multiple imports
- Each `import_tana_paste` call should ideally be under 10000 chars
