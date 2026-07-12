# DJEN — Operando em produção

Tudo aqui vem de operação real (consulta interativa por processo + sync
diário por OAB monitorada, 2026).

## Geo-bloqueio: o erro que só aparece no deploy

A API do Comunica responde **403 para IPs de origem fora do Brasil**.
Consequência traiçoeira: tudo funciona no seu notebook (IP brasileiro) e
falha em produção, porque seu backend está em `us-central1`/`us-east-1`/
Europa. Foi exatamente assim que descobrimos — smoke test de produção com
403 onde o teste local dava 200.

- **Rode o backend em região brasileira**: GCP `southamerica-east1` (São
  Paulo), AWS `sa-east-1`, Azure `brazilsouth`.
- Plataformas serverless/edge com egress imprevisível (Vercel, Cloudflare
  Workers, etc.): confirme a região de saída antes de confiar.
- Se o resto do seu app vive em outra região, isole a integração DJEN num
  serviço/função na região brasileira. Detalhe de SDK que custou horas: com
  Firebase/GCP, o *cliente* precisa apontar explicitamente para a região da
  function (`getFunctions(app, "southamerica-east1")`) — senão a chamada
  resolve para a região default e falha com "not found".
- Latência cruzada é real mas administrável (ex.: função em São Paulo
  escrevendo num Firestore multi-região US ≈ 120ms/write) — paralelize
  escritas em lotes pequenos (usamos 20 concorrentes) em vez de sequencial.
- O DataJud **não** tem geo-bloqueio; não generalize entre as duas APIs.

## Rate limit não documentado

Não há chave nem limite publicado, mas o limite existe: sob rajada a API
passa a responder **HTTP 500**. O que funcionou:

- **Espaçamento fixo de ~500ms** entre consultas sequenciais (inclusive
  entre as 7 variantes de sufixo de uma mesma OAB).
- **Timeout de 15s** por request (a API é bem mais rápida que o DataJud;
  15s é folga, não aperto).
- **Retry:** 1 retentativa com backoff fixo (~1,5s) para timeout e 5xx.
  **4xx não se retenta** — é erro de parâmetro (ou geo-bloqueio 403), e
  repetir não muda nada.
- **Circuit breaker:** 3 timeouts **consecutivos** abortam o run inteiro
  (API fora do ar; insistir só queima billing). Atenção à definição de
  "consecutivo": sucesso zera o contador; erro não-timeout (ex.: 500)
  também zera — 500 esporádico é o rate limit trabalhando, não indisponibilidade.

## Tetos de segurança (protegem seu run, não a API)

- **Teto por página/consulta:** o `count` é capado em ~10.000; nosso loop
  usa `MAX_PAGINAS = 200` × 50 itens como teto duro equivalente.
- **Teto de negócio por OAB/dia** (usamos 500): um advogado de alto volume
  (ou uma OAB digitada errada que casa com meio tribunal) não pode consumir
  o run inteiro. Excedente é logado e sinalizado — nunca cortado em
  silêncio, senão o monitoramento "parece ok" enquanto perde publicações.
- **Teto de duração do run** (usamos 25min) abaixo do timeout duro da
  plataforma, com margem para o flush final.

## Sanitização do HTML das publicações

O `texto` vem como HTML livre de dezenas de tribunais — na prática, input
não confiável renderizado dentro do seu app. Além do óbvio
(`<script>`, `<style>`, `<iframe>`, `<object>`, `<embed>`, handlers `on*`,
URLs `javascript:`), a lição não-óbvia:

**Remova também `class`, `style` e `id`.** Sanitizadores padrão (inclusive o
do Angular) preservam `class` por design. Se seu app usa CSS utilitário
global (Tailwind e afins), uma publicação contendo
`class="fixed inset-0 z-50"` **cobre a tela inteira do seu app** — UI
redressing/clickjacking sem uma linha de JavaScript.

Checklist do sanitizador (aplicado na exibição; armazene o HTML original):

- Allowlist de tags estruturais (`p`, `br`, `strong`, `em`, `table`…);
- Strip de `script`/`style`/`iframe`/`object`/`embed` **com o conteúdo**;
- Remoção de todos os atributos `on*`, `javascript:` em `href`/`src`, e de
  `class`/`style`/`id`;
- Links externos com `rel="noopener noreferrer"` e `target="_blank"`;
- Renderize dentro de um container com estilos isolados/neutros.

## Desenho de um monitoramento por OAB (o que funcionou)

- **Agendamento:** varredura principal de manhã (06:00, após a
  disponibilização do dia à 00:00) + **repescagem** à noite (22:00) apenas
  para quem a manhã não cobriu (checkpoint por dia). Repescagem não alerta.
- **Checkpoint por alvo monitorado por dia** (`lastSyncAt = "2026-07-12"`):
  runs abortados (breaker/teto) retomam só o que faltou.
- **Agrupe alvos repetidos**: se dois usuários monitoram a mesma OAB+UF,
  consulte uma vez e distribua o resultado — cada consulta ao DJEN custa 7
  variantes.
- **Kill switch remoto** (flag em config/banco lida no início do run) para
  desligar o sync sem deploy, e allowlist de usuários para rollout gradual.
- **Alertas:** falha total (0 processados com erros > 0) alerta só no run da
  manhã; exception não tratada = alerta crítico. Falha individual = warn
  agregado (o padrão é o mesmo do DataJud — ver skill `datajud`).
- **Digest para o usuário:** publicações "órfãs" (que não casam com nenhum
  processo cadastrado) vêm **primeiro** no e-mail — são processos fora do
  radar; citações antes de intimações. Garanta GC/limpeza de pendências
  antigas (ex.: 7 dias) para o digest não crescer para sempre.
- **Consulta manual vs. digest:** publicação capturada em consulta manual
  dentro da janela do dia ainda deve entrar no digest do titular da OAB
  (quem consultou não é necessariamente o intimado); histórico antigo
  importado nasce "já lido" para não inundar o primeiro digest.

## Aviso obrigatório na interface

O DJEN é fonte oficial, mas seu app não é: exiba de forma permanente que o
monitoramento é **ferramenta de auxílio e não substitui a consulta oficial**
ao diário/sistema do tribunal. Contagem de prazo continua sendo
responsabilidade do advogado — deixe isso escrito onde o usuário decide.
