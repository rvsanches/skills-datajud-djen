# DJEN — Montando consultas

## Endpoints

```
GET https://comunicaapi.pje.jus.br/api/v1/comunicacao
GET https://comunicaapi.pje.jus.br/api/v1/comunicacao/{hash}/certidao
```

- Sem autenticação (só o fluxo de *envio* pelos tribunais exige credencial).
- **Requisito de rede:** IP de origem brasileiro — de fora do Brasil a API
  responde **403** (detalhes em [producao.md](producao.md)).
- `Accept: application/json`.

## Filtros da consulta (query string)

| Parâmetro | Formato | Observações |
|---|---|---|
| `numeroOab` | string | Match **exato** — ver "Variantes de OAB" abaixo |
| `ufOab` | `SP`, `RJ`… | Combine sempre com `numeroOab` |
| `numeroProcesso` | 20 dígitos sem máscara | |
| `siglaTribunal` | `TJSP`, `TRF3`… | Útil para particionar volumes grandes |
| `dataDisponibilizacaoInicio` | `AAAA-MM-DD` | |
| `dataDisponibilizacaoFim` | `AAAA-MM-DD` | |
| `pagina` | inteiro, começa em 1 | |
| `itensPorPagina` | **máx. 50** | Acima disso: `items: []` silencioso |

Resposta: `{ "count": <total de resultados>, "items": [ ... ] }`.
Campos dos itens em [resposta.md](resposta.md).

## Paginação correta (com os quirks)

Três comportamentos reais que quebram a paginação ingênua:

1. `itensPorPagina > 50` → `items: []` com `count` preenchido, **sem erro**.
2. Página vazia no meio da sequência é (quase sempre) transiente — se você
   acumulou menos que `count`, retente a mesma página com backoff.
3. `count` aparenta ser capado em **10.000**; não conte com paginar além
   disso.

Loop validado em produção (JavaScript):

```js
const PAGE_SIZE = 50;        // máximo real da API
const MAX_PAGINAS = 200;     // 200 × 50 = teto de 10k, alinhado ao cap do count
const MAX_RETRIES_PAGINA = 2;

async function consultarComunicacoes(params, { timeoutMs = 15000 } = {}) {
  const itens = [];
  let count = null;
  let retriesPagina = 0;

  for (let pagina = 1; pagina <= MAX_PAGINAS; pagina++) {
    const query = new URLSearchParams({
      ...params,
      itensPorPagina: String(PAGE_SIZE),
      pagina: String(pagina),
    });
    const response = await fetchComTimeout(
        `https://comunicaapi.pje.jus.br/api/v1/comunicacao?${query}`,
        { headers: { Accept: "application/json" } }, timeoutMs);
    if (!response.ok) throw new Error(`DJEN respondeu ${response.status}`);

    const data = await response.json();
    if (count === null) count = data.count || 0;
    const pageItems = Array.isArray(data.items) ? data.items : [];

    if (pageItems.length === 0) {
      if (itens.length >= count) break;            // fim legítimo
      if (retriesPagina >= MAX_RETRIES_PAGINA) {
        return { itens, count, completo: false };  // desistiu — logue o delta
      }
      retriesPagina++;
      pagina--;                                    // repete a MESMA página
      await new Promise((r) => setTimeout(r, 1000 * retriesPagina));
      continue;
    }

    retriesPagina = 0;
    itens.push(...pageItems);
    if (itens.length >= count) break;
  }
  return { itens, count: count || 0, completo: itens.length >= (count || 0) };
}
```

O retorno `completo: false` importa: silenciosamente coletar menos do que o
`count` anunciado faz seu monitoramento "parecer ok" enquanto perde
publicações. Logue o delta.

## Variantes de OAB (o achado que mais importa)

O filtro `numeroOab` é comparado como **string exata**, e cada tribunal
registra a inscrição com um sufixo diferente: `123456` num tribunal,
`123456-O` (originária), `123456-A` (suplementar/transferência) em outros.
Medimos na prática: consultando só o número puro, **~90% das publicações de
uma OAB real ficavam invisíveis**.

Consulta completa por advogado = uma consulta por variante:

```js
const SUFIXOS_OAB = ["", "-O", "-A", "-N", "-B", "-S", "-E"];
function variantesOab(numero) {
  const base = String(numero).replace(/\D/g, ""); // só dígitos
  return SUFIXOS_OAB.map((s) => `${base}${s}`);
}
// 7 consultas por OAB monitorada; espace-as (~500ms) — ver producao.md
```

Para deduplicar/casar advogados entre publicações, normalize a OAB para só
dígitos (`numeroOabNormalizado`) e compare junto com a UF. **Nunca** use o
nome do advogado como chave — a grafia varia entre tribunais e até entre
publicações do mesmo tribunal.

## Janela de datas

- Publicações de um dia são disponibilizadas a partir das 00:00 daquele dia
  (fuso de Brasília); um job diário às 06:00 já captura o dia corrente.
- Use janela **ontem + hoje** (2 dias) em vez de só hoje: tolera atrasos de
  disponibilização e itens que entram fora de hora. A sobreposição gera
  repetidos — deduplique pelo `id` (ver [resposta.md](resposta.md)).
- Calcule as datas no fuso `America/Sao_Paulo`, não no UTC do runtime — às
  22h em Brasília, UTC já virou o dia seguinte e sua "janela de hoje" fica
  errada. Em JS: `new Intl.DateTimeFormat("en-CA", { timeZone:
  "America/Sao_Paulo" }).format(new Date())` → `AAAA-MM-DD`.

## Certidão em PDF

Toda comunicação tem um `hash`; a certidão oficial correspondente sai em:

```
https://comunicaapi.pje.jus.br/api/v1/comunicacao/{hash}/certidao
```

É um endpoint público — no frontend, basta abrir o link em nova aba (usuário
também precisa estar em IP brasileiro). Não há necessidade de proxy no
backend.
