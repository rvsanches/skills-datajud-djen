# DJEN — Interpretando respostas

## Formato geral

```json
{ "count": 428, "items": [ { ... }, { ... } ] }
```

`count` é o total de resultados do filtro (aparenta ser capado em ~10.000);
`items` é a página corrente. Página vazia com `count > acumulado` é
transiente — ver o loop de paginação em [consultas.md](consultas.md).

## Campos de cada item

A API mistura **snake_case e camelCase** nos nomes de campo — não assuma um
padrão único; copie os nomes exatamente como abaixo (observados em
produção):

| Campo na API | Tipo | Conteúdo |
|---|---|---|
| `id` | number | Identificador único da comunicação — **use como chave de dedupe** |
| `hash` | string | Identificador para a certidão PDF (`/api/v1/comunicacao/{hash}/certidao`) |
| `numero_processo` | string | Número CNJ, 20 dígitos sem máscara |
| `numeroprocessocommascara` | string | Número CNJ com máscara (tudo minúsculo, sem separadores no nome!) |
| `siglaTribunal` | string | `TJSP`, `TRF3`… (camelCase) |
| `tipoComunicacao` | string | `Intimação`, `Citação`, `Ato ordinatório`… |
| `tipoDocumento` | string | `Despacho`, `Sentença`, `Decisão`… |
| `nomeOrgao` | string | Vara/órgão julgador |
| `nomeClasse` | string | Classe processual |
| `data_disponibilizacao` | string | `AAAA-MM-DD` |
| `meio` / `meiocompleto` | string | Meio de publicação (sigla / descrição) |
| `texto` | string | **HTML bruto** da publicação — tratar como input hostil |
| `link` | string | URL do documento no tribunal |
| `status` | string | Status da publicação (valores não catalogados por completo) |
| `motivo_cancelamento` | string\|null | Preenchido quando o tribunal cancela a publicação |
| `destinatarios` | array | `[{ "nome": ..., "polo": ... }]` — partes |
| `destinatarioadvogados` | array | `[{ "advogado": { "nome": ..., "numero_oab": ..., "uf_oab": ... } }]` — note o aninhamento em `.advogado` |

Exemplo ilustrativo (anonimizado; campos menos relevantes omitidos):

```json
{
  "id": 661000001,
  "hash": "aBcDeFgHiJkLmNoPqRsTuVwXyZ01234",
  "numero_processo": "00000012320268260100",
  "numeroprocessocommascara": "0000001-23.2026.8.26.0100",
  "siglaTribunal": "TJSP",
  "tipoComunicacao": "Intimação",
  "tipoDocumento": "Despacho",
  "nomeOrgao": "1ª Vara Cível de São Paulo",
  "nomeClasse": "Procedimento Comum Cível",
  "data_disponibilizacao": "2026-07-11",
  "texto": "<p>Vistos. Intime-se a parte autora para...</p>",
  "link": "https://...",
  "motivo_cancelamento": null,
  "destinatarios": [{ "nome": "FULANO DE TAL", "polo": "A" }],
  "destinatarioadvogados": [
    { "advogado": { "nome": "ADVOGADA EXEMPLO", "numero_oab": "123456", "uf_oab": "SP" } }
  ]
}
```

Como o esquema não é formalmente versionado, trate campos como opcionais e
tolere valores novos (ex.: novos `tipoComunicacao`).

## Validação mínima e normalização

Regras que usamos antes de persistir:

- **Descarte** itens sem `id` ou sem `numero_processo` — sem eles não há
  dedupe nem vínculo com processo.
- Normalize para o seu padrão de nomes na entrada (ex.: tudo camelCase), e
  normalize a OAB de cada advogado para só dígitos
  (`numeroOabNormalizado = numero_oab.replace(/\D/g, "")`) — é essa forma
  que se compara com a OAB monitorada, nunca a string crua (que pode vir com
  sufixo `-O`/`-A`…).
- Guarde o `hash` mesmo que não use de imediato — é o que gera a certidão.
- `texto` pode ser muito grande. Se seu armazenamento tem limite por
  registro (ex.: Firestore ~1 MiB/doc), trunque preservando o **início** (é
  onde está o comando da intimação) com um marcador de truncamento, e mantenha
  o `link` para o teor completo. Usamos teto de 200.000 caracteres.

## Dedupe

- Chave natural: `id` (number). Janela de consulta sobreposta (ontem+hoje,
  recomendada) e variantes de OAB retornam a mesma comunicação várias vezes.
- Padrão que funcionou sem transações: ID de documento **determinístico**
  derivado do `id` (ex.: `djen_{id}` — ou `djen_{id}_{tenant}` em app
  multi-tenant onde a mesma publicação pode interessar a clientes
  diferentes) + escrita *create-only*, tratando "já existe" como sucesso
  esperado.

## Cancelamento de publicação

Tribunais **cancelam** publicações após disponibilizá-las. Uma comunicação
que você já capturou pode reaparecer na API com `motivo_cancelamento`
preenchido (e `status` alterado). Consequências:

- O upsert *create-only* precisa de uma exceção: quando o item chega com
  `motivo_cancelamento`, atualize o registro existente (status + motivo).
- Exiba a publicação como "Cancelada" em vez de apagá-la — o usuário pode
  já ter visto a versão original e precisa saber que ela caiu.

## Vinculando a processos (matching)

Para casar publicações com uma carteira de processos, compare o
`numero_processo` normalizado para 20 dígitos (aceite entrada com e sem
máscara do seu lado). Publicação que não casa com nenhum processo conhecido
não é lixo: em monitoramento por OAB, ela é justamente o alerta de processo
novo/fora do radar — trate "órfãs" como cidadãs de primeira classe.
