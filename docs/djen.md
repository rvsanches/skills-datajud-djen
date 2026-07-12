# API do DJEN / Comunica PJe — guia de campo

> Versão para leitura humana. O conteúdo completo e detalhado vive na skill
> [`skills/djen`](../skills/djen/SKILL.md) e suas referências — este guia é
> o resumo com contexto e histórias.

O DJEN (Diário de Justiça Eletrônico Nacional, Resolução CNJ nº 455)
centraliza intimações, citações e publicações de todos os tribunais
brasileiros. A API pública do Comunica PJe
(`https://comunicaapi.pje.jus.br`) permite consultá-las **sem autenticação**
— por processo, por OAB do advogado, por tribunal e por data. Se o DataJud
mostra o que aconteceu no processo, o DJEN mostra **o que dispara prazo**.

Este guia documenta o que aprendemos implementando monitoramento de
publicações em produção num SaaS jurídico (Judis).

## O essencial em 30 segundos

```bash
# precisa rodar de IP brasileiro!
curl 'https://comunicaapi.pje.jus.br/api/v1/comunicacao?numeroOab=123456&ufOab=SP&dataDisponibilizacaoInicio=2026-07-11&dataDisponibilizacaoFim=2026-07-12&pagina=1&itensPorPagina=50' \
  -H 'Accept: application/json'
```

Resposta: `{ "count": <total>, "items": [ ... ] }`. Certidão oficial em PDF:
`GET /api/v1/comunicacao/{hash}/certidao`.

## As lições que custaram caro

**1. A API geo-bloqueia IPs fora do Brasil (HTTP 403).** Funciona no seu
notebook, falha no deploy — porque sua função/servidor está em
`us-central1`. Descobrimos num smoke test de produção. Rode o backend em
região brasileira (GCP `southamerica-east1`, AWS `sa-east-1`). O DataJud não
tem esse bloqueio; é específico do Comunica.

**2. `itensPorPagina` máximo é 50, e o erro é silencioso.** Peça 100 e a API
responde `items: []` com `count` preenchido — parece "sem resultados", mas é
o parâmetro.

**3. Sem os sufixos de OAB, ~90% das publicações ficam invisíveis.** O
filtro `numeroOab` é match exato de string, e cada tribunal grava a
inscrição de um jeito: `123456`, `123456-O`, `123456-A`… Monitoramento sério
consulta as 7 variantes (`"", -O, -A, -N, -B, -S, -E`) e normaliza a OAB
para dígitos na comparação. E nunca case por nome de advogado — a grafia
varia entre tribunais.

**4. Página vazia não é fim dos dados.** Sob instabilidade, páginas vazias
aparecem no meio da paginação. Se o acumulado é menor que o `count`, retente
a mesma página com backoff. E o `count` aparenta ser capado em 10.000 — para
volume, particione por data/tribunal.

**5. Existe rate limit, só não está documentado.** Sob rajada a API responde
500. Espaçamento de ~500ms entre consultas + retry único com backoff + circuit
breaker resolveram; 500 esporádico é o rate limit trabalhando, não
indisponibilidade.

**6. O `texto` das publicações é HTML hostil — inclusive o `class`.**
Sanitizadores padrão preservam `class` por design. Com Tailwind global, uma
publicação com `class="fixed inset-0 z-50"` cobre a tela inteira do app
(UI redressing). Remova `class`/`style`/`id` além do arsenal usual
(`script`, handlers `on*`, `javascript:`).

**7. Tribunais cancelam publicações depois de publicá-las.** Uma comunicação
já capturada pode reaparecer com `motivo_cancelamento` preenchido. Atualize
o registro e exiba "Cancelada" — não apague nem ignore.

## Mapa do conteúdo completo

| Documento | Conteúdo |
|---|---|
| [`SKILL.md`](../skills/djen/SKILL.md) | Visão geral e as 7 regras de ouro |
| [`references/consultas.md`](../skills/djen/references/consultas.md) | Filtros, paginação correta (com código), variantes de OAB, janelas de data, certidão |
| [`references/resposta.md`](../skills/djen/references/resposta.md) | Todos os campos (snake_case + camelCase), dedupe, cancelamento, matching com processos |
| [`references/producao.md`](../skills/djen/references/producao.md) | Geo-bloqueio e regiões, retry/breaker, tetos de segurança, sanitização de HTML, agendamento, digest e alertas |
