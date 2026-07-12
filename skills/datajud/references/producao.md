# DataJud — Operando em produção

Tudo aqui vem de operação real (sync 2×/dia de centenas de processos +
consulta interativa, 2025-2026). O tema constante: **a API é cronicamente
saturada** — a chave é compartilhada nacionalmente e o cluster vive no
limite. Projete para lentidão e indisponibilidade como estado normal.

## Latência real (medida)

| Cenário | Comportamento |
|---|---|
| Alias específico, cluster ok | Poucos segundos |
| Alias específico, cluster degradado | 10-35s, com sucessos aos 41,5s |
| Curinga `api_publica_*` | 504 após ~60s, de qualquer origem (0/27 sucessos medidos) |
| Qualquer request > ~60s | O gateway do CNJ corta com 504 — não espere mais que isso |
| Datacenter cloud vs IP residencial | Sem diferença relevante para o DataJud (não há geo-bloqueio — ao contrário do DJEN) |

## Timeouts recomendados

- **Fluxo interativo** (usuário esperando na tela): ~35s por tentativa, com
  orçamento total de ~60-70s incluindo retries. Timeout de 15s ou menos
  estoura o tempo todo — não é rede ruim, é a API.
- **Batch/sync**: ~75s por tentativa. Valores maiores não ajudam: o gateway
  corta aos ~60s; o excedente é só margem para jitter.
- Implemente timeout de verdade (`AbortController` no fetch do Node — o fetch
  nativo não tem timeout próprio).

Histórico que motivou esses números: começamos com 20s (estourava sempre),
passamos por 40s, 60s e 120s até entender que o gateway corta aos ~60s e
que o problema real era o curinga — com alias por tribunal, 75s cobre com
folga.

## Política de retry

```
429                  → retry com backoff curto (ex.: 2s, 4s). A rejeição da
                       fila do ES responde em ~0,1s, então errar é barato.
200 parcial sem hit  → retry com backoff (o shard pode voltar); esgotado,
                       reporte indisponibilidade, NUNCA "não encontrado".
504                  → NÃO retente. Cada tentativa custa ~60s e indica
                       saturação sistêmica. Deixe para o circuit breaker.
Timeout (Abort)      → não retente na mesma passada; conte para o breaker.
4xx (exceto 429)     → não retente; é bug seu (número inválido, alias errado,
                       chave rotacionada → 401/403).
```

Use um **orçamento de tempo** além do contador de tentativas: não inicie
retry se o tempo total decorrido já passou do budget (respostas parciais
lentas de 10-35s cada consomem o teto da sua function/job rapidamente).

## Circuit breaker (essencial em batch)

Aborte o run inteiro após **3 falhas lentas consecutivas** — onde falha
lenta = timeout **ou** 504. Sucesso zera o contador.

Por que contar o 504: aprendemos da pior forma. O breaker original contava
só timeouts; quando o CNJ passou a responder 504 aos 60s (em vez de
silêncio), o breaker nunca disparava e o job "queimava" ~19 minutos de
billing falhando processo a processo. Falha rápida (429, 4xx) não conta —
ela não indica que a API está fora, e o breaker dispararia à toa.

## Rate limiting próprio

A cota é global por chave compartilhada — você protege a todos (inclusive
de si mesmo):

- **Interativo:** limite por usuário (usamos 10 consultas/min). Um bug de
  retry em loop no cliente drena a cota de todo o país.
- **Batch:** espaçamento fixo entre chamadas (usamos 2s).
- **Valide antes de chamar:** número com 20 dígitos e alias derivável.
  Request inválido também consome cota.

## Desenho de um sync periódico (o que funcionou)

- **Horário:** evite o "horário cheio" — às 05:00 concentram-se os syncs de
  todo mundo contra o CNJ. Rodamos às **06:00** (varredura principal) e
  **22:00** (repescagem: só o que ficou para trás, ex.: `lastSync > 20h`).
  A repescagem não dispara alertas — senão vira ruído diário.
- **Fila por desatualização:** processe ordenado por `lastSyncedAt`
  ascendente (mais desatualizado primeiro) com paginação server-side no seu
  banco. Assim, um run abortado (breaker, teto de tempo) retoma naturalmente
  do ponto certo no run seguinte.
- **Checkpoint só em sucesso:** grave `lastSyncedAt` apenas quando a
  consulta funcionou — falha fica no início da fila para o próximo run.
  Exceção: número inválido/sem alias grava o checkpoint mesmo assim (senão
  ele entope o início da fila para sempre).
- **Teto de duração:** limite o run (usamos 25min) abaixo do timeout duro da
  plataforma, com margem para o commit final.
- **Não re-sincronize fases terminais** (encerrado, desistido) — só consome
  cota.

## Erros e alertas (o que aprendemos a alertar)

- **Falha individual de processo = warn, não error.** Quando a API cai, um
  `error` por processo vira spam de alertas. Quem alerta é o **agregado do
  run**.
- **Alerta de falha total** (`processados == 0 && erros > 0`) apenas no run
  principal da manhã. Significa API fora do ar **ou chave rotacionada** — a
  mensagem do alerta deve mandar conferir
  <https://datajud-wiki.cnj.jus.br/api-publica/acesso/> antes de qualquer
  debug.
- **Erro fatal do job** (exception não tratada) = alerta crítico. Sem isso,
  um sync silenciosamente quebrado leva dias para ser notado.
- **No fluxo interativo**, distinga as mensagens: 429 persistente / 504 /
  resposta parcial → "API do CNJ sobrecarregada, tente mais tarde";
  não encontrado confiável → "confira o número ou cadastre manualmente".
  Induzir o usuário a "verificar o número" quando a API está fora gera
  suporte à toa.

## Incidente ilustrativo: a chave rotacionada

Runs de dois dias seguidos com 100% de timeout. Nada tinha mudado no código.
Causa: o CNJ rotacionou a chave pública (a anterior valia desde meses antes)
e a antiga passou a falhar. Correções derivadas:

1. Chave em secret versionado (troca sem deploy).
2. Alerta de falha total aponta explicitamente para a página da wiki onde a
   chave atual é publicada.
3. Runbook: falha total repentina → conferir chave **antes** de debugar
   código.
