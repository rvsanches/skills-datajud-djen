---
name: datajud
description: >-
  Guia prático da API Pública do DataJud (CNJ) — consulta de metadados e
  movimentações de processos judiciais brasileiros via Elasticsearch em
  api-publica.datajud.cnj.jus.br. Vai além da documentação oficial: registra
  conhecimento de produção como a consulta obrigatória pelo alias do tribunal
  (o curinga api_publica_* retorna 504), respostas 200 parciais que simulam
  "não encontrado", timeouts realistas (a API responde em 10-40s), política de
  retry para 429, circuit breaker, datas de 14 dígitos em horário de Brasília
  e códigos TPU de movimentos. Use sempre que for integrar, consultar, depurar
  ou otimizar chamadas ao DataJud — busca de processo por número CNJ,
  sincronização de andamentos/movimentações processuais, erros 504/429/timeout
  ou montagem de queries Elasticsearch contra tribunais brasileiros.
---

# API Pública do DataJud (CNJ)

O DataJud é a base nacional de metadados processuais do CNJ. A API Pública
(`https://api-publica.datajud.cnj.jus.br`) expõe esses dados via Elasticsearch
Search API: um índice por tribunal, consultas em Query DSL.

O que ela devolve: metadados do processo (classe, órgão julgador, grau,
data de ajuizamento, assuntos) e a lista de **movimentações** (andamentos).
O que ela **não** devolve: teor de decisões, intimações ou publicações — para
comunicações que disparam prazo, use a API do DJEN (skill `djen`).

Documentação oficial: <https://datajud-wiki.cnj.jus.br/api-publica/>.
Este guia registra o que a wiki não conta, aprendido operando a API em
produção (projeto Judis, 2025-2026).

## Início rápido

```bash
curl -X POST 'https://api-publica.datajud.cnj.jus.br/api_publica_tjsp/_search' \
  -H 'Authorization: APIKey <CHAVE_PUBLICA>' \
  -H 'Content-Type: application/json' \
  -d '{"query": {"match": {"numeroProcesso": "00000012320268260100"}}}'
```

- A chave pública está em <https://datajud-wiki.cnj.jus.br/api-publica/acesso/>.
  O esquema do header é literalmente `APIKey` (não `Bearer`).
- O número do processo vai **sem máscara**: 20 dígitos.
- O alias (`api_publica_tjsp`) é derivável do próprio número CNJ — ver abaixo.

## As 7 regras de ouro (aprendidas em produção)

1. **Consulte sempre pelo alias do tribunal, nunca pelo curinga.**
   `api_publica_*` faz fan-out em ~90 índices, não termina dentro do limite de
   ~60s do gateway do CNJ e retorna **504 de qualquer origem** (medido: 0/27
   sucessos em 2026-07). O alias específico responde em 10-35s mesmo com o
   cluster degradado. O alias é derivável dos dígitos J.TR do número CNJ —
   função pronta em [references/consultas.md](references/consultas.md).

2. **Dimensione timeouts para dezenas de segundos, não para poucos.**
   A API é cronicamente saturada: respostas de sucesso levando 10-40s são
   normais (medido: 200 OK aos 41,5s). Timeout de 15s "estoura sempre".
   Valores que funcionam: ~35s por tentativa em fluxo interativo, ~75s em
   batch. Acima de ~60s o gateway corta com 504 — esperar mais que isso por
   uma mesma tentativa não traz resposta.

3. **Retry para 429; nunca para 504.** O 429 (`es_rejected_execution_exception`)
   é uma rejeição rápida (~0,1s) da fila de busca — retry com backoff curto é
   barato e costuma passar. O 504 custa ~60s por tentativa e indica saturação
   sistêmica — deixe para o circuit breaker.

4. **HTTP 200 não significa resposta completa.** Sob saturação o
   Elasticsearch responde `200` com `_shards.failed > 0` e, se o processo
   estava num shard rejeitado, `total = 0`. **"Não encontrado" só é confiável
   quando `_shards.failed == 0`.** Já exibimos "processo não encontrado" para
   processo existente por ignorar isso. Detalhes em
   [references/resposta.md](references/resposta.md).

5. **Datas de 14 dígitos estão em horário de Brasília (UTC-3), sem marcação.**
   `dataHora: "20260710143000"` é 14h30 em Brasília. Parsear ingenuamente num
   runtime UTC desloca tudo em 3h — errado para horário de audiência. E o
   mesmo campo pode vir como epoch de 13 dígitos ou ISO, dependendo do
   tribunal.

6. **A chave é pública, compartilhada e rotacionada sem aviso.** A cota é
   global por chave (você compete com todo o Brasil). Valide o número CNJ
   antes de chamar (não desperdice cota com requests inválidos), aplique rate
   limit próprio, e trate falha total repentina como possível rotação de
   chave — confira a wiki antes de debugar seu código.

7. **Indexação atrasa.** Processos recém-distribuídos podem levar semanas
   para aparecer. "Não encontrado" (confiável) não significa que o processo
   não existe — ofereça cadastro manual como fallback.

## Referências (leia conforme a tarefa)

| Arquivo | Quando ler |
|---|---|
| [references/consultas.md](references/consultas.md) | Montar requests: autenticação, derivação do alias por número CNJ (função completa), queries Elasticsearch, paginação |
| [references/resposta.md](references/resposta.md) | Interpretar respostas: estrutura do `_source`, parsing defensivo, datas, códigos TPU de movimentos → fase processual |
| [references/producao.md](references/producao.md) | Operar em produção: timeouts, retry, circuit breaker, rate limiting, agendamento de sync, detecção de novidade, monitoramento |

## Relação com o DJEN

| | DataJud | DJEN (skill `djen`) |
|---|---|---|
| Conteúdo | Movimentações — *o que aconteceu* | Intimações/citações — *o que dispara prazo* |
| Autenticação | Chave pública (header `APIKey`) | Nenhuma |
| Geo-bloqueio | Não tem (latência é igual de fora do BR) | **403 para IP fora do Brasil** |
| Protocolo | POST, Elasticsearch Query DSL | GET, REST com query string |

Ambas são do CNJ, sem SLA nem versionamento formal — projete para
indisponibilidade e mudanças sem aviso.
