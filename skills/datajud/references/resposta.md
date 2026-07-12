# DataJud — Interpretando respostas

## Estrutura da resposta

Resposta padrão da Search API do Elasticsearch. Exemplo ilustrativo
(anonimizado e abreviado):

```json
{
  "took": 12034,
  "timed_out": false,
  "_shards": { "total": 5, "successful": 5, "skipped": 0, "failed": 0 },
  "hits": {
    "total": { "value": 1, "relation": "eq" },
    "hits": [
      {
        "_index": "processos_tjsp_...",
        "_source": {
          "numeroProcesso": "00000012320268260100",
          "tribunal": "TJSP",
          "grau": "G1",
          "dataAjuizamento": "2026-01-15T00:00:00.000Z",
          "classe": { "codigo": 7, "nome": "Procedimento Comum Cível" },
          "assuntos": [{ "codigo": 1234, "nome": "..." }],
          "orgaoJulgador": {
            "codigo": 13597,
            "nome": "1ª Vara Cível",
            "codigoMunicipioIBGE": 3550308
          },
          "movimentos": [
            { "codigo": 51, "nome": "Conclusão", "dataHora": "20260710143000" }
          ]
        }
      }
    ]
  }
}
```

Campos do `_source` que usamos em produção:

| Campo | Conteúdo | Observações |
|---|---|---|
| `numeroProcesso` | 20 dígitos, sem máscara | |
| `tribunal` | Sigla (`TJSP`, `TRF3`…) | |
| `grau` | `G1`, `G2`… | |
| `dataAjuizamento` | Data de ajuizamento | Formato varia — ver "Datas" abaixo |
| `classe.codigo` / `classe.nome` | Classe processual (TPU) | |
| `orgaoJulgador.*` | Vara/órgão | `codigoMunicipioIBGE` é o código IBGE do município, não a UF |
| `movimentos[]` | Andamentos | Ver seção própria |

O esquema **varia por tribunal** (os índices são alimentados por sistemas
diferentes). Trate todo campo como opcional e não assuma tipos além do
observado.

## Parsing defensivo — as 3 armadilhas

**1. `total > 0` sem array `hits`.** A API ocasionalmente responde malformada:
`hits.total.value > 0` mas sem `hits.hits` (ou sem `_source` no primeiro
item). Acessar `hits.hits[0]._source` direto causa `TypeError`. Valide a
cadeia inteira:

```js
const hits = data.hits;
const encontrado = Boolean(
  hits && hits.total && hits.total.value > 0 &&
  Array.isArray(hits.hits) && hits.hits[0] && hits.hits[0]._source
);
```

**2. Resposta 200 parcial (`_shards.failed > 0`).** Sob saturação, o
Elasticsearch responde `200 OK` tendo consultado só parte dos shards. Se o
processo estava num shard rejeitado, `total = 0` — e o seu app conclui
"processo não encontrado" **falsamente** (aconteceu em produção: processo
existente no TJMT, com 21 movimentos, reportado como inexistente na tela de
cadastro). Regra:

```js
const parcial = Boolean(data._shards && data._shards.failed > 0);
const temHit = Boolean(data.hits?.total?.value > 0);
// parcial && !temHit  → NÃO é "não encontrado"; é indisponibilidade.
//                       Retente com backoff; esgotado, reporte "API
//                       sobrecarregada, tente mais tarde".
// parcial && temHit   → pode usar: o _source do hit encontrado é íntegro.
```

**3. "Não encontrado" legítimo ainda pode ser atraso de indexação.**
Processos recém-distribuídos levam dias a semanas para aparecer no DataJud.
Não trate ausência como inexistência — ofereça caminho manual.

## Datas: três formatos, um deles perigoso

O mesmo campo de data (`dataHora` de movimento, `dataAjuizamento`) chega em
formatos diferentes conforme o tribunal:

1. **14 dígitos `YYYYMMDDHHmmss`** — está em **horário de Brasília (UTC-3,
   sem horário de verão desde 2019), sem indicação de fuso**. Este é o
   formato perigoso: `new Date(2026, 6, 10, 14, 30)` num runtime UTC (Cloud
   Functions, Lambda, containers) desloca o instante em 3h — errado para
   horário de audiência. Monte o instante com offset explícito.
2. **13 dígitos** — epoch em milissegundos.
3. **String ISO** — parse direto.

Função validada em produção (retorna ISO UTC ou `null`):

```js
function formatarDataHora(raw) {
  if (!raw) return null;
  const s = String(raw);
  if (/^\d{14}$/.test(s)) {
    const iso = `${s.substring(0, 4)}-${s.substring(4, 6)}-` +
      `${s.substring(6, 8)}T${s.substring(8, 10)}:` +
      `${s.substring(10, 12)}:${s.substring(12, 14)}-03:00`;
    const parsed = new Date(iso);
    return isNaN(parsed.getTime()) ? null : parsed.toISOString();
  }
  if (!isNaN(Number(s)) && s.length === 13) {
    return new Date(Number(s)).toISOString();
  }
  const parsed = new Date(s);
  return isNaN(parsed.getTime()) ? null : parsed.toISOString();
}
// "20240115093000" → "2024-01-15T12:30:00.000Z"
```

Na exibição, se você armazenou ISO UTC, formate com fuso explícito
(ex.: Angular `date:'dd/MM/yyyy HH:mm':'UTC'` ou convertendo para
`America/Sao_Paulo`) para não deslocar de novo.

## Movimentos e códigos TPU

Cada item de `movimentos[]` tem `dataHora`, `nome` e o código da Tabela
Processual Unificada (TPU). **O aninhamento varia por tribunal**: às vezes
`codigo`/`nome` vêm direto no movimento, às vezes dentro de
`movimentoNacional.{codigo, nome}`. Leia ambos:

```js
const cod = mov.movimentoNacional ? mov.movimentoNacional.codigo : mov.codigo;
const nome = mov.nome || (mov.movimentoNacional && mov.movimentoNacional.nome)
  || "Movimentação registrada";
```

Os movimentos **não vêm garantidamente ordenados** — ordene por `dataHora`
desc antes de usar.

### Inferindo a fase processual

Mapeamento validado em produção (percorra do movimento mais recente para o
mais antigo e pare no primeiro match; combine código TPU com substring do
nome, porque nem todo tribunal preenche o código):

| Fase | Códigos TPU | Substrings no nome (lowercase) |
|---|---|---|
| Encerrado | 22, 246, 861, 269 | "arquivamento definitivo", "baixa definitiva", "extinção da execução", "transito em julgado" |
| Desistido | 466, 200, 149 | "desistência", "homologada a transação", "homologado o acordo" |
| Suspenso | 25, 314 | "suspensão do processo", "suspenso o processo", "suspensão da execução" |
| Sobrestado | 11009, 12154 | "sobrestamento", "sobrestado o recurso", "sobrestado o processo" |
| Concluso | 51 | "conclusão", "concluso" |
| Em Andamento | — | (default, nenhum match) |

Consulta oficial da TPU: <https://www.cnj.jus.br/sgt/consulta_publica_movimentos.php>.

## Detectando novidade entre syncs

Comparar só a `dataHora` do movimento mais recente perde casos reais: novo
evento na mesma data, reordenação, movimento removido/corrigido pelo
tribunal. Use uma assinatura do conjunto inteiro:

```js
function assinaturaMovimentos(movimentos) {
  if (!Array.isArray(movimentos)) return "";
  return movimentos
      .map((m) => `${(m && m.dataHora) || ""}|${(m && m.nome) || ""}`)
      .join("\n");
}
// Há novidade quando a assinatura mudou E a nova lista não está vazia
// (lista vazia da API não deve apagar o que você já tem).
```

Se o usuário pode registrar movimentos manuais junto aos automáticos,
etiquete a origem de cada um (`origem: "datajud" | "manual"`) e, no sync,
substitua apenas os automáticos, preservando os manuais no merge.
