# Monitoramento de Preços Amazon

Este repositório contém o fluxo do n8n utilizado para monitorar preços de produtos na Amazon.

## Visão Geral

O workflow recebe uma lista de links via webhook, visita cada página, coleta o título e o preço de cada produto e envia um relatório único por e-mail.

## Passo a Passo

1. **Webhook**
   - Recebe do formulário (ex.: Tally) o nome, e-mail e uma lista de links da Amazon.

2. **CodeExtractLinks** (node `Code`, modo: *Run Once for Each Item*)
   - Divide os links recebidos em múltiplos items, um para cada link.
   - Exemplo de código:

```javascript
const links = $json.body.data.fields
  .find(f => f.label.includes('links'))
  .value.split('\n')
  .map(l => l.trim())
  .filter(Boolean);
return links.map(link => ({ json: { link } }));
```

3. **HTTP Request**
   - Busca o HTML de cada página de produto da Amazon.
   - Método: `GET`
   - URL: `{{$json.link}}`
   - Headers incluem um `User-Agent` de navegador para evitar bloqueio.

4. **Merge** (Combine By Position)
   - Junta cada item do HTTP Request (campo `data` com o HTML) ao item original contendo o `link`.

5. **Code1** (node `Code`, modo: *Run Once for Each Item*)
   - Extrai título e preço do HTML da página.
   - Código resumido:

```javascript
const html = $json.data || "";
const link = $json.link;

if (!html) {
  return { json: { titulo: "HTML vazio ou inválido", preco: "Preço não encontrado", link } };
}

let titulo = "Título não encontrado";
const mTitle = html.match(/<meta name="title" content="(.*?)"/i) ||
               html.match(/<title>(.*?)<\/title>/i);
if (mTitle && mTitle[1]) {
  titulo = mTitle[1].replace(" | Amazon.com.br", "").trim();
}

let preco = "Preço não encontrado";
const mMetaPreco = html.match(/<meta name="twitter:data1" content="(.*?)"/i);
if (mMetaPreco && mMetaPreco[1]) {
  preco = mMetaPreco[1];
} else {
  const mFallback = html.match(/R\$[\s]*[\d.,]+/);
  if (mFallback) preco = mFallback[0];
}

return { json: { link, titulo, preco } };
```

6. **AggregateResults** (node `Code`, modo: *Run Once for All Items*)
   - Junta todos os itens em um único array `resultados`.
   - Código:

```javascript
const resultados = $items("Code1").map(item => ({
  titulo: item.json.titulo,
  preco: item.json.preco,
  link: item.json.link
}));

return [{ json: { resultados } }];
```

7. **Gmail → Send a Message**
   - Envia um único e-mail para o usuário com todos os produtos monitorados.
   - Exemplo de corpo do e-mail:

```
Olá,

Segue abaixo o relatório de preços monitorados em {{ new Date().toLocaleString('pt-BR') }}:

{{ $json.resultados
    .map(item => `- ${item.titulo}: ${item.preco} (Link: ${item.link})`)
    .join("\n") }}

Abraço,
Alvo Certo Monitoramento
```

---

Em uma frase: *"Este fluxo recebe uma lista de links da Amazon via webhook, extrai o título e preço de cada produto, agrega os resultados em um único item e envia por e-mail um relatório formatado."*

