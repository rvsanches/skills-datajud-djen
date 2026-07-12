# DataJud — Montando consultas

## Endpoint e autenticação

```
POST https://api-publica.datajud.cnj.jus.br/{alias}/_search
Authorization: APIKey <CHAVE_PUBLICA>
Content-Type: application/json
```

- `{alias}` identifica o tribunal (ex.: `api_publica_tjsp`). Catálogo oficial:
  <https://datajud-wiki.cnj.jus.br/api-publica/endpoints/>.
- A chave é **pública e compartilhada** — publicada em
  <https://datajud-wiki.cnj.jus.br/api-publica/acesso/>. O esquema do header é
  `APIKey` (literal, não `Bearer`). Guarde-a em secret mesmo assim: o CNJ a
  rotaciona sem aviso e você vai querer trocá-la sem redeploy de código.
- Não há endpoint de "consulta nacional" utilizável: o curinga
  `api_publica_*` retorna 504 na prática (fan-out em ~90 índices estoura o
  limite de ~60s do gateway). Sempre resolva o tribunal primeiro.

## Derivando o alias a partir do número CNJ

O número unificado (Res. CNJ 65/2008) tem o formato
`NNNNNNN-DD.AAAA.J.TR.OOOO`. Nos 20 dígitos sem máscara:

- **J** (14º dígito, índice 13): segmento do Judiciário
- **TR** (15º-16º dígitos): tribunal dentro do segmento

Justiça Estadual e Eleitoral numeram as UFs em ordem alfabética. Atenção ao
Distrito Federal: o código TR é `07`, mas os aliases usam `dft`
(`api_publica_tjdft`, `api_publica_tre-dft`).

Função pronta (JavaScript, sem dependências) — validada em produção:

```js
// TR → UF (Res. CNJ 65/2008: UFs em ordem alfabética; DF usa "dft").
const TR_UF = {
  "01": "ac", "02": "al", "03": "ap", "04": "am", "05": "ba", "06": "ce",
  "07": "dft", "08": "es", "09": "go", "10": "ma", "11": "mt", "12": "ms",
  "13": "mg", "14": "pa", "15": "pb", "16": "pr", "17": "pe", "18": "pi",
  "19": "rj", "20": "rn", "21": "rs", "22": "ro", "23": "rr", "24": "sc",
  "25": "se", "26": "sp", "27": "to",
};

/**
 * Deriva o alias do índice DataJud a partir do número CNJ (20 dígitos).
 * Retorna null quando o segmento não tem índice no DataJud
 * (STF, CNJ, CJMs, número inválido).
 */
function aliasDatajud(numeroLimpo) {
  if (!/^\d{20}$/.test(numeroLimpo)) return null;
  const j = numeroLimpo[13];
  const tr = numeroLimpo.substring(14, 16);
  const n = Number(tr);
  const uf = TR_UF[tr];
  if (j === "8") return uf ? `api_publica_tj${uf}` : null;
  if (j === "4") return n >= 1 && n <= 6 ? `api_publica_trf${n}` : null;
  if (j === "5") {
    if (tr === "00") return "api_publica_tst";
    return n >= 1 && n <= 24 ? `api_publica_trt${n}` : null;
  }
  if (j === "6") {
    if (tr === "00") return "api_publica_tse";
    return uf ? `api_publica_tre-${uf}` : null;
  }
  if (j === "3") return tr === "00" ? "api_publica_stj" : null;
  if (j === "7") return tr === "00" ? "api_publica_stm" : null;
  if (j === "9") {
    if (uf === "mg" || uf === "rs" || uf === "sp") {
      return `api_publica_tjm${uf}`;
    }
    return null;
  }
  return null; // J=1 (STF) e J=2 (CNJ) não integram o DataJud.
}
```

Resumo dos segmentos:

| J | Segmento | Alias |
|---|---|---|
| 1 | STF | — (não integra o DataJud) |
| 2 | CNJ | — (não integra o DataJud) |
| 3 | STJ | `api_publica_stj` (TR=00) |
| 4 | Justiça Federal | `api_publica_trf1`…`trf6` |
| 5 | Justiça do Trabalho | `api_publica_tst` (TR=00), `api_publica_trt1`…`trt24` |
| 6 | Justiça Eleitoral | `api_publica_tse` (TR=00), `api_publica_tre-{uf}` |
| 7 | Justiça Militar da União | `api_publica_stm` (TR=00) |
| 8 | Justiça Estadual | `api_publica_tj{uf}` (DF = `tjdft`) |
| 9 | Justiça Militar Estadual | `api_publica_tjmmg`, `api_publica_tjmrs`, `api_publica_tjmsp` |

Quando `aliasDatajud` retorna `null`, não chame a API: o request seria
inválido e ainda consumiria a cota global da chave.

## Query: busca por número de processo

O caso de uso central. Limpe a máscara e valide 20 dígitos **antes** de
chamar:

```js
const cleanProcesso = numeroProcesso.replace(/\D/g, "");
if (cleanProcesso.length !== 20) throw new Error("Número CNJ inválido");
```

Corpo da requisição:

```json
{ "query": { "match": { "numeroProcesso": "00000012320268260100" } } }
```

O número CNJ é único no Judiciário, então o resultado esperado é 0 ou 1 hit —
leia `hits.hits[0]._source` (com o parsing defensivo descrito em
[resposta.md](resposta.md)). Um mesmo processo pode aparecer em índices de
tribunais diferentes em grau de recurso; consultando pelo alias derivado do
número você obtém o processo no tribunal de origem.

## Outras queries (Query DSL)

O endpoint é a Search API do Elasticsearch, então a wiki documenta o Query
DSL padrão — `match`, `range`, `bool`, paginação com `size`/`from` e
`search_after` + `sort` para varreduras longas. Exemplo de busca por classe e
órgão julgador:

```json
{
  "size": 100,
  "query": {
    "bool": {
      "must": [
        { "match": { "classe.codigo": 1116 } },
        { "match": { "orgaoJulgador.codigo": 13597 } }
      ]
    }
  },
  "sort": [{ "@timestamp": { "order": "asc" } }]
}
```

**Aviso de produção:** nosso uso intensivo é o lookup por número de processo.
Varreduras amplas (`match_all`, agregações, `search_after` em milhares de
páginas) rodam sobre um cluster cronicamente saturado e uma cota global
compartilhada — espere lentidão, 429 e respostas parciais com muito mais
frequência, e espaçe as chamadas de forma agressiva (ver
[producao.md](producao.md)).

## Rate limit e cota

- A cota é **global por API key** — e a chave é a mesma para todo mundo.
  Não há limite documentado por IP; na prática você compete com todos os
  consumidores do Brasil, o que explica a saturação crônica.
- Imponha seu próprio rate limit por usuário/rotina (em produção usamos
  10 consultas/min por usuário no fluxo interativo e 2s entre chamadas em
  batch) para que um bug em loop não drene a disponibilidade de todos.
