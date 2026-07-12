# API Pública do DataJud — guia de campo

> Versão para leitura humana. O conteúdo completo e detalhado vive na skill
> [`skills/datajud`](../skills/datajud/SKILL.md) e suas referências —
> este guia é o resumo com contexto e histórias.

O DataJud é a base nacional de metadados processuais do CNJ. A
[API Pública](https://datajud-wiki.cnj.jus.br/api-publica/) expõe, via
Elasticsearch, os metadados e as movimentações de processos de quase todos
os tribunais do país. É uma API valiosa — e cronicamente saturada, com
comportamentos que a documentação oficial não descreve. Este guia documenta
o que aprendemos operando-a em produção num SaaS jurídico (Judis), com sync
diário de centenas de processos e consulta interativa.

## O essencial em 30 segundos

```bash
curl -X POST 'https://api-publica.datajud.cnj.jus.br/api_publica_tjsp/_search' \
  -H 'Authorization: APIKey <CHAVE_PUBLICA>' \
  -H 'Content-Type: application/json' \
  -d '{"query": {"match": {"numeroProcesso": "00000012320268260100"}}}'
```

- Chave pública (compartilhada por todos) na
  [wiki do CNJ](https://datajud-wiki.cnj.jus.br/api-publica/acesso/); o
  esquema do header é `APIKey`, não `Bearer`.
- Um índice por tribunal (`api_publica_tjsp`, `api_publica_trf3`…), corpo em
  Elasticsearch Query DSL, número de processo sem máscara (20 dígitos).

## As lições que custaram caro

**1. O curinga `api_publica_*` morreu — consulte pelo alias do tribunal.**
A consulta nacional (fan-out em ~90 índices) não termina dentro do limite de
~60s do gateway e retorna 504 praticamente sempre (medimos 0/27 sucessos, de
três origens de rede diferentes). A boa notícia: o alias é derivável dos
dígitos `J.TR` do próprio número CNJ — publicamos a função pronta em
[`consultas.md`](../skills/datajud/references/consultas.md).

**2. A API é lenta por natureza — dimensione timeouts em dezenas de segundos.**
Sucessos aos 30-40s são rotina (medimos 200 OK aos 41,5s). Timeout de 15s
falha quase sempre; usamos ~35s por tentativa no fluxo interativo e 75s em
batch. Acima de ~60s o gateway corta com 504, então esperar mais não ajuda.

**3. HTTP 200 pode ser resposta parcial — e "não encontrado" vira mentira.**
Sob saturação, o Elasticsearch responde 200 com `_shards.failed > 0`. Se o
processo estava num shard rejeitado, `total = 0`. Exibimos "processo não
encontrado" para um processo existente até aprender: **só confie no "não
encontrado" quando `_shards.failed == 0`**.

**4. Retry para 429, circuit breaker para 504.** O 429 é rejeição rápida
(~0,1s) — retry barato com backoff curto. O 504 custa ~60s por tentativa;
retryar é queimar dinheiro. Em batch, 3 falhas lentas consecutivas (timeout
ou 504) devem abortar o run.

**5. Datas de 14 dígitos vêm em horário de Brasília, sem dizer isso.**
`"20260710143000"` é 14h30 **UTC-3**. Runtime em UTC (Cloud Functions,
Lambda) parseia errado e desloca audiências em 3 horas. E o mesmo campo pode
vir como epoch de 13 dígitos ou ISO, conforme o tribunal.

**6. A chave é de todos — e o CNJ a rotaciona sem aviso.** Tivemos dois dias
de 100% de falha até descobrir que a chave tinha sido trocada na wiki.
Guarde-a em secret (troca sem deploy), monitore falha total e confira a wiki
antes de debugar seu código.

## Mapa do conteúdo completo

| Documento | Conteúdo |
|---|---|
| [`SKILL.md`](../skills/datajud/SKILL.md) | Visão geral e as 7 regras de ouro |
| [`references/consultas.md`](../skills/datajud/references/consultas.md) | Autenticação, derivação de alias por número CNJ (função completa), queries, cota |
| [`references/resposta.md`](../skills/datajud/references/resposta.md) | Estrutura do `_source`, parsing defensivo, datas, códigos TPU → fase processual, detecção de novidade |
| [`references/producao.md`](../skills/datajud/references/producao.md) | Latências medidas, timeouts, retry, circuit breaker, rate limiting, desenho de sync, alertas |
