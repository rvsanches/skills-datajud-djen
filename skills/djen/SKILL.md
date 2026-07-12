---
name: djen
description: >-
  Guia prático da API do DJEN — Diário de Justiça Eletrônico Nacional
  (Comunica PJe, comunicaapi.pje.jus.br), a API pública do CNJ para
  intimações, citações e publicações processuais de todos os tribunais
  brasileiros. Vai além da documentação oficial: registra conhecimento de
  produção como o geo-bloqueio (403 para IPs fora do Brasil — derruba cloud
  functions em regiões estrangeiras), o limite silencioso de 50 itens por
  página, os sufixos de inscrição da OAB que variam por tribunal (sem eles
  ~90% das publicações ficam invisíveis), páginas vazias transientes, teto de
  10.000 resultados e sanitização do HTML das publicações. Use sempre que for
  integrar, consultar ou depurar o DJEN/Comunica PJe — monitoramento de
  publicações por OAB, busca de intimações por processo, certidões em PDF ou
  erros 403/500/páginas vazias nessa API.
---

# API do DJEN / Comunica PJe (CNJ)

O DJEN (Diário de Justiça Eletrônico Nacional) centraliza as comunicações
processuais — **intimações, citações e demais publicações** — de todos os
tribunais brasileiros (Resolução CNJ nº 455; cobertura nacional obrigatória
desde 2025). A API pública do Comunica PJe expõe essas comunicações para
consulta.

Diferença-chave para o DataJud: o DataJud mostra **movimentações** (o que
aconteceu no processo); o DJEN mostra **o que dispara prazo** contra o
advogado. Para monitorar prazos, é o DJEN que importa.

Portal: <https://comunica.pje.jus.br>. Swagger: <https://comunicaapi.pje.jus.br>.
Este guia registra o que a documentação não conta, aprendido operando a API
em produção (projeto Judis, 2026).

## Início rápido

```bash
# ATENÇÃO: precisa rodar de IP brasileiro (ver regra nº 1)
curl 'https://comunicaapi.pje.jus.br/api/v1/comunicacao?numeroOab=123456&ufOab=SP&dataDisponibilizacaoInicio=2026-07-11&dataDisponibilizacaoFim=2026-07-12&pagina=1&itensPorPagina=50' \
  -H 'Accept: application/json'
```

Sem autenticação, sem chave, sem cadastro. A resposta é
`{ "count": <total>, "items": [ ... ] }`.

## As 7 regras de ouro (aprendidas em produção)

1. **A API geo-bloqueia IPs fora do Brasil: HTTP 403.** Funciona do seu
   notebook e falha no deploy — porque sua cloud function/container está em
   região estrangeira (`us-central1`, `us-east-1`…). Rode o backend em região
   brasileira (GCP `southamerica-east1`, AWS `sa-east-1`). Descobrimos em
   smoke test de produção: mesma requisição, 200 do Brasil, 403 dos EUA.
   O DataJud **não** tem esse bloqueio — é específico do Comunica.

2. **`itensPorPagina` máximo é 50 — e o erro é silencioso.** Acima de 50 a
   API responde `items: []` com `count` preenchido, sem erro. Parece "sem
   resultados", mas é o parâmetro. Use exatamente 50 e pagine.

3. **Página vazia ≠ fim dos dados.** Sob instabilidade a API retorna páginas
   intermitentemente vazias no meio da paginação. Se `acumulado < count`,
   retente a **mesma** página com backoff antes de desistir.

4. **Monitorando por OAB, consulte as variantes de sufixo.** O filtro
   `numeroOab` é match exato de string, e cada tribunal grava a inscrição de
   um jeito (`123456` no TJSP, `123456-O` ou `123456-A` no TJMT…). Sem varrer
   os sufixos `["", "-O", "-A", "-N", "-B", "-S", "-E"]`, a maior parte das
   publicações fica invisível. E **nunca** case por nome do advogado —
   grafias variam entre tribunais; só OAB normalizada (dígitos) + UF.

5. **`count` é capado em ~10.000** por consulta. Para volumes grandes,
   particione por data/tribunal em vez de paginar até o fim.

6. **Deduplique pelo campo `id`** (número, único por comunicação). A mesma
   consulta em janelas sobrepostas retorna repetidos — janela sobreposta é
   inclusive recomendada (ontem+hoje) para tolerar atrasos de
   disponibilização.

7. **O campo `texto` é HTML de dezenas de tribunais — trate como input
   hostil.** Sanitize na exibição removendo não só `script`/handlers, mas
   também `class`/`style`/`id`: com CSS utilitário global (Tailwind), um
   `class="fixed inset-0 z-50"` vindo numa publicação cobre a tela inteira
   do seu app (UI redressing).

## Referências (leia conforme a tarefa)

| Arquivo | Quando ler |
|---|---|
| [references/consultas.md](references/consultas.md) | Montar requests: filtros, paginação correta (com os quirks), variantes de OAB, certidão em PDF, janelas de data |
| [references/resposta.md](references/resposta.md) | Interpretar respostas: todos os campos (snake_case misturado com camelCase), dedupe, cancelamento de publicação, normalização |
| [references/producao.md](references/producao.md) | Operar em produção: geo-bloqueio e regiões, retry/backoff/circuit breaker, tetos de segurança, sanitização de HTML, agendamento e alertas |

## Relação com o DataJud

| | DJEN | DataJud (skill `datajud`) |
|---|---|---|
| Conteúdo | Intimações/citações — *o que dispara prazo* | Movimentações — *o que aconteceu* |
| Autenticação | Nenhuma | Chave pública (header `APIKey`) |
| Geo-bloqueio | **403 para IP fora do Brasil** | Não tem |
| Protocolo | GET REST com query string | POST, Elasticsearch Query DSL |
| Rate limit | Não documentado (500 sob rajada) | Cota global por chave compartilhada |

Ambas são do CNJ, sem SLA nem versionamento formal — projete para
indisponibilidade e mudanças sem aviso, e mantenha na interface o aviso de
que a ferramenta **não substitui a consulta oficial** ao diário.
