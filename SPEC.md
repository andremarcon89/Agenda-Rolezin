# Agenda Rolezin — Especificação (SPEC)

> Status: **rascunho para aprovação** · Versão 0.1 · 27/10/2026
> Nada foi implementado. As seções marcadas com **[DECISÃO]** dependem de escolha do casal (consolidadas na §9).

---

## 1. Visão geral

Aplicação web pessoal, privada, hospedada em nuvem, para que duas pessoas (contas separadas, dados compartilhados) organizem a programação cultural de São Paulo:

1. **Coletar** programação a partir de URLs, PDFs, texto colado e cadastro manual, com extração assistida por LLM e **fila de revisão humana obrigatória** antes de publicar.
2. **Triar** no *Cardápio Cultural* (filtros + ações "Quero ir", "Não tenho interesse", "Ingresso comprado").
3. **Visualizar** em dois calendários alternáveis: *Cardápio de Possibilidades* (tudo que está disponível) e *Encontro Marcado* (só o que foi escolhido).
4. **Exportar** o *Encontro Marcado* como feed ICS assinável.

### 1.1 Fora de escopo (v1)

- Qualquer acesso público, compartilhamento com terceiros ou cadastro aberto.
- Compra de ingressos ou integração com bilheterias.
- App nativo (será PWA instalável).
- Notificações push próprias (lembretes ficam a cargo do calendário que assina o ICS).
- Multi-"casa"/multi-tenant: há **um único grupo implícito** com dois membros.

---

## 2. Stack proposta

### 2.1 Recomendação: **Next.js (App Router, TypeScript) + Supabase + Vercel**

| Camada | Escolha | Por quê |
|---|---|---|
| Front + back | Next.js 15+ (App Router), TypeScript, React Server Components, Route Handlers | Um único projeto para UI e API; ótimo suporte da Vercel; ecossistema maduro. |
| UI | Tailwind CSS + shadcn/ui (Radix) | Componentes acessíveis, mobile-first, sem dependência de biblioteca pesada. |
| Banco | Supabase Postgres | Postgres de verdade: `generate_series` (expansão de janelas), `pg_trgm` + `unaccent` (deduplicação), `daterange`/GiST, funções SQL, RLS. |
| Autenticação | Supabase Auth (cadastro desabilitado; 2 contas criadas manualmente) | Pronto, gratuito, integrado à RLS. |
| Arquivos | Supabase Storage (bucket privado) | PDFs de catálogos e eventuais comprovantes de ingresso; upload direto do navegador via URL assinada (contorna limite de corpo das funções da Vercel). |
| Hospedagem | Vercel (plano Hobby) | Deploy por `git push`, previews por branch, HTTPS, uso pessoal não comercial permitido. |
| LLM | API da Anthropic via SDK oficial `@anthropic-ai/sdk`, *structured outputs* | Extração para JSON validado por schema (Zod). Ver §6. |
| Datas | `date-fns` + `@date-fns/tz` | Fuso fixo `America/Sao_Paulo`. |
| HTML → texto | `@mozilla/readability` + `linkedom`; leitura prévia de JSON-LD `schema.org/Event` | Reduz tokens; JSON-LD dá dados estruturados sem custo de LLM quando existir. |
| PDF | Envio nativo de PDF à API (com divisão por páginas via `pdf-lib`) e/ou extração de texto (`pdfjs-dist`) | Ver §6.3. |
| ICS | Geração própria (texto RFC 5545) ou `ics` (npm) | Feed simples; poucas linhas de código. |
| Testes | Vitest (unidade), Playwright (e2e em viewport de celular) | Playwright já roda bem em CI e localmente. |
| Migrações | Supabase CLI (`supabase/migrations/*.sql`) + tipos gerados (`supabase gen types`) | SQL como fonte de verdade; tipos TS derivados. |

### 2.2 Prós e contras

**Prós**
- Custo zero de infraestrutura no uso previsto (2 usuários, volume baixo).
- Postgres resolve nativamente as partes mais difíceis do domínio (recorrência virtual, similaridade de títulos, intervalos de datas).
- RLS garante "nada público" na camada do banco, não só na aplicação.
- Um repositório, uma linguagem (TS + SQL).

**Contras / pontos de atenção**
- **Pausa do Supabase gratuito por inatividade** (projetos free são pausados após ~1 semana sem uso). Mitigação: *keepalive* diário (cron da Vercel chamando um endpoint que faz um `select 1`), ou aceitar reativar manualmente. **[DECISÃO]**
- **Limites das funções da Vercel Hobby** (duração máxima por invocação, corpo de requisição ~4,5 MB, cron com frequência limitada). A extração de PDFs grandes precisa ser fatiada em partes, com estado persistido no banco (§6.4). Verificar os limites vigentes na Fase 0.
- Acoplamento a dois fornecedores (Vercel e Supabase). Mitigado por usar Postgres padrão e Next.js, que rodam em qualquer lugar.

### 2.3 Alternativas consideradas

| Alternativa | Vantagem | Por que não é a primeira opção |
|---|---|---|
| Next.js + **Neon** (Postgres) + Auth.js + Vercel | Neon não "pausa" o projeto (apenas *scale-to-zero* com *cold start* curto) | Autenticação, storage e RLS ficam por nossa conta; mais código para manter. Boa opção B se a pausa do Supabase incomodar. |
| SvelteKit ou Remix no lugar do Next.js | Menos complexidade conceitual que RSC | Ganho pequeno; ecossistema de componentes menor. |
| Cloudflare Pages/Workers + D1 (SQLite) | Free tier generoso, sem pausa | Sem `pg_trgm`, sem RLS, recorrência e dedupe teriam de ir para o código; auth própria. |
| PocketBase em VPS/Fly.io | Binário único, simples | Exige operar servidor; SQLite; backups por nossa conta. |
| Planilha (Google Sheets) + Apps Script | Zero infraestrutura | Não sustenta o modelo de dados (ocorrências, proveniência, dedupe) nem a UX de calendário. |

---

## 3. Conceitos do domínio

| Conceito | Definição |
|---|---|
| **Programa** (obra) | O "quê": um filme, uma peça, um show, uma exposição, uma oficina. Tem título, tipo, ficha etc. Independe de quando/onde acontece. |
| **Coleção** | Agrupador opcional de programas: festival, mostra, ciclo (ex.: "50ª Mostra Internacional de Cinema"). |
| **Local** | O "onde": nome, endereço, bairro, coordenadas opcionais. |
| **Ocorrência** | O "quando". Dois tipos: |
| ↳ **Sessão** (pontual) | Data e hora de início definidas (ex.: 14/11 às 20h). Pode ser cadastrada avulsa ou **gerada por uma temporada**. |
| ↳ **Janela** (contínua) | Intervalo de datas + horário de funcionamento por dia da semana (ex.: 01/11–15/12, ter–dom 10h–18h). **Não gera registros por dia**: é expandida virtualmente nas consultas. |
| **Temporada** | Regra de recorrência (ex.: qui–sáb 20h, dom 18h, de 06/11 a 14/12) que é **materializada em sessões** (finitas, dezenas de registros). |
| **Escolha** | Decisão do casal: no nível do programa (interesse/descartado) ou de uma sessão/visita (agendado/ingresso comprado/descartado). |
| **Fonte** | De onde veio a informação (URL, PDF, texto, manual), com snapshot do conteúdo usado. |
| **Item extraído** | Resultado bruto da extração por LLM, que aguarda revisão (rascunho → aprovado/rejeitado). |

**Por que temporada é materializada e janela é virtual?** Uma temporada gera poucas sessões discretas, cada uma com sentido próprio (pode-se comprar ingresso para *aquela* sessão, ela pode ser cancelada, ter preço diferente). Uma janela representa "aberto neste horário", sem unidades discretas; materializá-la geraria um registro por dia sem informação nova. Para escolher uma ida a uma exposição, cria-se uma **visita** (data + hora pretendida) apontando para a janela.

---

## 4. Esquema do banco de dados

Convenções: nomes em português, `snake_case`, sem acentos; chaves `uuid` (`gen_random_uuid()`); carimbos `timestamptz`; datas "de calendário" (`date`/`time`) interpretadas sempre em `America/Sao_Paulo`.

### 4.1 Extensões e utilitários

```sql
create extension if not exists pg_trgm;
create extension if not exists unaccent;

-- Normalização para comparação (minúsculas, sem acento, sem pontuação, espaços colapsados).
-- unaccent com dicionário explícito para poder ser IMMUTABLE e usado em colunas geradas/índices.
create or replace function normalizar(t text) returns text
language sql immutable parallel safe as $$
  select trim(regexp_replace(
           regexp_replace(lower(public.unaccent('public.unaccent'::regdictionary, coalesce(t, ''))),
                          '[^a-z0-9 ]', ' ', 'g'),
           '\s+', ' ', 'g'))
$$;

create type tipo_atividade as enum
  ('cinema','teatro','danca','musica','exposicao','literatura','outros');
create type status_revisao  as enum ('rascunho','aprovado','rejeitado');
create type status_escolha  as enum ('interesse','agendado','ingresso_comprado','descartado');
create type tipo_fonte      as enum ('url','pdf','texto','imagem','manual');
create type status_extracao as enum ('pendente','processando','concluida','erro');
```

### 4.2 Membros e segurança

```sql
create table membro (
  id         uuid primary key references auth.users(id) on delete cascade,
  nome       text not null,          -- "André", "<nome dela>"
  cor        text not null,          -- cor de identificação na UI
  criado_em  timestamptz not null default now()
);

-- Verdadeiro se o usuário autenticado é um dos membros.
create or replace function eh_membro() returns boolean
language sql stable security definer set search_path = public as $$
  select exists (select 1 from membro where id = auth.uid())
$$;
```

**Regra geral de RLS:** todas as tabelas de domínio têm RLS ligada e uma única política `for all using (eh_membro()) with check (eh_membro())`. Não existe política para `anon`. O endpoint ICS (sem sessão) usa a *service role* no servidor, com consulta restrita ao dono do token (§7).

### 4.3 Locais e coleções

```sql
create table local (
  id                uuid primary key default gen_random_uuid(),
  nome              text not null,
  nome_normalizado  text generated always as (normalizar(nome)) stored,
  endereco          text,
  bairro            text,               -- texto livre com autocomplete a partir dos já usados
  regiao            text,               -- opcional: 'Centro','Zona Oeste'... [DECISÃO]
  cidade            text not null default 'São Paulo',
  latitude          numeric(9,6),
  longitude         numeric(9,6),
  site              text,
  observacoes       text,
  criado_em         timestamptz not null default now(),
  atualizado_em     timestamptz not null default now()
);
create index local_nome_trgm on local using gin (nome_normalizado gin_trgm_ops);
create index local_bairro    on local (bairro);

-- Apelidos para casar variações vindas das fontes ("CCSP" = "Centro Cultural São Paulo").
create table local_alias (
  alias_normalizado text primary key,
  local_id          uuid not null references local(id) on delete cascade
);

create table colecao (
  id           uuid primary key default gen_random_uuid(),
  nome         text not null,           -- "50ª Mostra Internacional de Cinema de SP"
  data_inicio  date,
  data_fim     date,
  site         text,
  criado_em    timestamptz not null default now()
);
```

### 4.4 Programas

```sql
create table programa (
  id                  uuid primary key default gen_random_uuid(),
  titulo              text not null,
  titulo_original     text,
  titulo_normalizado  text generated always as (normalizar(titulo)) stored,
  tipo                tipo_atividade not null,
  subtipo             text,                 -- 'show', 'stand-up', 'oficina', 'debate'...
  descricao           text,
  ficha               jsonb not null default '{}',
     -- ex.: {"direcao":["…"],"elenco":["…"],"dramaturgia":"…","pais":"BR","ano":2026,"artistas":["…"]}
  duracao_min         int check (duracao_min > 0),
  classificacao       text,                 -- 'L','10','12','14','16','18'
  gratuito            boolean,              -- padrão do programa; null = não informado
  preco_min           numeric(8,2),
  preco_max           numeric(8,2),
  preco_obs           text,                 -- "meia R$ 20; grátis às quartas"
  links               jsonb not null default '[]',   -- [{"rotulo":"Site","url":"…"}]
  imagem_url          text,
  colecao_id          uuid references colecao(id) on delete set null,
  mesclado_em         uuid references programa(id),  -- se não nulo, este programa foi absorvido por outro
  criado_por          uuid references membro(id),
  criado_em           timestamptz not null default now(),
  atualizado_em       timestamptz not null default now()
);
create index programa_titulo_trgm on programa using gin (titulo_normalizado gin_trgm_ops);
create index programa_tipo        on programa (tipo) where mesclado_em is null;
create index programa_colecao     on programa (colecao_id);
```

Observação: não há `status_revisao` em `programa`. **Só entra em `programa` o que foi aprovado**; rascunhos vivem em `item_extraido` (§4.7). Isso evita que toda consulta precise filtrar "aprovados" e impede que um rascunho vaze para os calendários.

### 4.5 Ocorrências

```sql
-- Regra de recorrência materializada em sessões.
create table temporada (
  id               uuid primary key default gen_random_uuid(),
  programa_id      uuid not null references programa(id) on delete cascade,
  local_id         uuid not null references local(id),
  data_inicio      date not null,
  data_fim         date not null,                 -- obrigatória; "tempo indeterminado" => horizonte (§4.5.1)
  regra            jsonb not null,
     -- [{"dias":[4,5,6],"hora":"20:00"},{"dias":[7],"hora":"18:00"}]   dias ISO: 1=seg … 7=dom
  datas_excluidas  date[] not null default '{}',
  duracao_min      int,
  gratuito         boolean,
  preco_min        numeric(8,2),
  preco_max        numeric(8,2),
  preco_obs        text,
  link_ingresso    text,
  fonte_id         uuid references fonte(id),
  criado_em        timestamptz not null default now(),
  atualizado_em    timestamptz not null default now(),
  check (data_fim >= data_inicio)
);

-- Sessão pontual (avulsa ou gerada por temporada).
create table sessao (
  id             uuid primary key default gen_random_uuid(),
  programa_id    uuid not null references programa(id) on delete cascade,
  local_id       uuid not null references local(id),
  temporada_id   uuid references temporada(id) on delete set null,
  inicio         timestamptz not null,
  fim            timestamptz,                 -- se nulo: inicio + duracao do programa (ou 2h)
  sala           text,
  versao         text,                        -- 'legendado','dublado','com Libras'…
  gratuito       boolean,                     -- sobrepõe o padrão do programa
  preco_min      numeric(8,2),
  preco_max      numeric(8,2),
  preco_obs      text,
  link_ingresso  text,
  cancelada      boolean not null default false,
  observacoes    text,
  fonte_id       uuid references fonte(id),
  criado_em      timestamptz not null default now(),
  atualizado_em  timestamptz not null default now(),
  check (fim is null or fim > inicio)
);
create index sessao_inicio            on sessao (inicio) where not cancelada;
create index sessao_programa_inicio   on sessao (programa_id, inicio);
create index sessao_local_inicio      on sessao (local_id, inicio);
create unique index sessao_unica      on sessao (programa_id, local_id, inicio);  -- dedupe exato

-- Ocorrência contínua (exposição, instalação, feira).
create table janela (
  id             uuid primary key default gen_random_uuid(),
  programa_id    uuid not null references programa(id) on delete cascade,
  local_id       uuid not null references local(id),
  data_inicio    date not null,
  data_fim       date not null,
  gratuito       boolean,
  preco_min      numeric(8,2),
  preco_max      numeric(8,2),
  preco_obs      text,                        -- "grátis aos sábados"
  link           text,
  observacoes    text,
  fonte_id       uuid references fonte(id),
  criado_em      timestamptz not null default now(),
  atualizado_em  timestamptz not null default now(),
  check (data_fim >= data_inicio)
);
create index janela_periodo on janela using gist (daterange(data_inicio, data_fim, '[]'));

create table janela_horario (
  janela_id   uuid not null references janela(id) on delete cascade,
  dia_semana  smallint not null check (dia_semana between 1 and 7),   -- ISO
  abre        time not null,
  fecha       time not null,
  primary key (janela_id, dia_semana, abre),
  check (fecha > abre)
);

-- Exceções pontuais: fechado num feriado, ou aberto em dia/horário extraordinário.
create table janela_excecao (
  janela_id  uuid not null references janela(id) on delete cascade,
  data       date not null,
  fechado    boolean not null default true,
  abre       time,          -- usados quando fechado = false (abertura/horário extraordinário)
  fecha      time,
  motivo     text,
  primary key (janela_id, data)
);
```

#### 4.5.1 Regras de geração de sessões a partir de temporada

- Expansão feita no servidor (TypeScript, função pura testada) ao salvar/alterar a temporada: para cada dia em `[data_inicio, data_fim]` fora de `datas_excluidas`, para cada item da regra cujo `dias` contenha o dia ISO, cria sessão em `data + hora` no fuso `America/Sao_Paulo`.
- **Reexpansão segura:** ao editar a regra, sessões futuras *sem escolha associada* são apagadas e regeneradas; sessões *com escolha* nunca são apagadas — se não existirem mais na nova regra, são marcadas `cancelada = true` e a UI alerta o casal.
- **Temporada sem data de término** ("em cartaz por tempo indeterminado"): grava-se `data_fim = hoje + 90 dias` e marca-se para reavaliação; não há expansão infinita.

### 4.6 Escolhas do casal

Uma tabela única cobre os três níveis de decisão:

| Nível | `sessao_id` | `janela_id` + `data_visita` | Status permitidos | Exemplo |
|---|---|---|---|---|
| Programa (triagem) | nulo | nulo | `interesse`, `descartado` | "Talvez" / "Não tenho interesse" no Cardápio |
| Sessão | preenchido | nulo | `agendado`, `ingresso_comprado`, `descartado` | "Quero ir" na sessão de sáb 20h |
| Visita a janela | nulo | preenchidos | `agendado`, `ingresso_comprado` | "Vamos à exposição dom 16/11 às 14h" |

```sql
create table escolha (
  id               uuid primary key default gen_random_uuid(),
  programa_id      uuid not null references programa(id) on delete cascade,
  sessao_id        uuid references sessao(id) on delete restrict,
  janela_id        uuid references janela(id) on delete restrict,
  data_visita      date,
  hora_visita      time,                       -- opcional; default de exibição 14h  [DECISÃO]
  status           status_escolha not null,
  participantes    uuid[] not null default '{}',   -- ids de membro; {a}, {b} ou {a,b}
  observacoes      text,
  valor_pago       numeric(8,2),
  codigo_ingresso  text,
  anexo_path       text,                       -- comprovante no Storage (opcional)
  marcado_por      uuid not null references membro(id),
  marcado_em       timestamptz not null default now(),
  atualizado_por   uuid references membro(id),
  atualizado_em    timestamptz not null default now(),
  check (num_nonnulls(sessao_id, janela_id) <= 1),
  check ((janela_id is null) = (data_visita is null)),
  check (
    (sessao_id is null and janela_id is null and status in ('interesse','descartado'))
    or (sessao_id is not null and status in ('agendado','ingresso_comprado','descartado'))
    or (janela_id is not null and status in ('agendado','ingresso_comprado'))
  )
);
create unique index escolha_programa on escolha (programa_id) where sessao_id is null and janela_id is null;
create unique index escolha_sessao   on escolha (sessao_id)   where sessao_id is not null;
create unique index escolha_visita   on escolha (janela_id, data_visita) where janela_id is not null;
create index        escolha_status   on escolha (status);

-- Histórico (para reversibilidade e "quem marcou e quando"), preenchido por trigger.
create table escolha_historico (
  id               bigint generated always as identity primary key,
  escolha_id       uuid not null,
  status_anterior  status_escolha,
  status_novo      status_escolha,          -- null = escolha removida
  alterado_por     uuid references membro(id),
  alterado_em      timestamptz not null default now(),
  snapshot         jsonb not null
);
```

- **"Quem vai" é armazenado de forma absoluta** (ids), não como "eu/ela": "eu" depende de quem está olhando. A UI traduz para "Eu / {nome} / Ambos" conforme o usuário logado.
- **Reversibilidade:** "Não tenho interesse" grava `descartado`; desfazer apaga a linha (ou volta ao status anterior via histórico). Há uma tela "Descartados" para recuperar itens.
- `ingresso_comprado` implica `agendado`: é um estado posterior, não paralelo.

### 4.7 Fontes, extração e revisão

```sql
create table fonte (
  id                  uuid primary key default gen_random_uuid(),
  tipo                tipo_fonte not null,
  url                 text,
  arquivo_path        text,                 -- caminho no Storage (PDF/imagem)
  nome_arquivo        text,
  titulo              text,
  texto_bruto         text,                 -- snapshot do conteúdo efetivamente enviado ao LLM
  conteudo_hash       text,                 -- sha256 do conteúdo, para detectar mudança/reenvio
  coletado_em         timestamptz not null default now(),
  coletado_por        uuid references membro(id),
  monitorar           boolean not null default false,   -- recoleta periódica (Fase 8)
  ultima_verificacao  timestamptz,
  check (tipo <> 'url' or url is not null),
  check (tipo not in ('pdf','imagem') or arquivo_path is not null)
);
create index fonte_url  on fonte (url);
create index fonte_hash on fonte (conteudo_hash);

create table extracao (
  id              uuid primary key default gen_random_uuid(),
  fonte_id        uuid not null references fonte(id) on delete cascade,
  parte           int not null default 1,     -- fatiamento (ex.: páginas 1–15 de um PDF)
  total_partes    int not null default 1,
  intervalo       text,                       -- "p. 1–15"
  status          status_extracao not null default 'pendente',
  modelo          text,
  tokens_entrada  int,
  tokens_saida    int,
  tokens_cache    int,
  custo_usd       numeric(10,4),
  erro            text,
  iniciado_em     timestamptz,
  concluido_em    timestamptz
);
create index extracao_pendente on extracao (status, fonte_id);

create table item_extraido (
  id               uuid primary key default gen_random_uuid(),
  extracao_id      uuid not null references extracao(id) on delete cascade,
  fonte_id         uuid not null references fonte(id) on delete cascade,
  payload          jsonb not null,     -- programa + local + ocorrências, no schema de extração (§6.2)
  trechos          jsonb not null default '[]',
     -- [{"campo":"sessoes[0].inicio","texto":"Sáb 15/11, 20h","verificado":true}]
  confianca        numeric(3,2),
  alertas          text[] not null default '{}',   -- "ano inferido", "local não encontrado"...
  candidatos_dup   jsonb not null default '[]',    -- [{"programa_id":"…","score":0.82,"motivo":"título+local"}]
  status_revisao   status_revisao not null default 'rascunho',
  programa_id      uuid references programa(id),  -- resultado após aprovar (novo ou mesclado)
  revisado_por     uuid references membro(id),
  revisado_em      timestamptz,
  motivo_rejeicao  text,
  criado_em        timestamptz not null default now()
);
create index item_fila on item_extraido (status_revisao, criado_em);

-- Proveniência: quais fontes embasam cada programa (N:N).
create table programa_fonte (
  programa_id       uuid not null references programa(id) on delete cascade,
  fonte_id          uuid not null references fonte(id) on delete cascade,
  item_extraido_id  uuid references item_extraido(id),
  primary key (programa_id, fonte_id)
);

-- Pares que o casal já declarou "não são o mesmo programa" (não sugerir de novo).
create table nao_duplicata (
  programa_a  uuid not null references programa(id) on delete cascade,
  programa_b  uuid not null references programa(id) on delete cascade,
  primary key (programa_a, programa_b),
  check (programa_a < programa_b)
);
```

Cadastro manual também cria uma `fonte` (`tipo = 'manual'`) para que toda informação tenha proveniência uniforme, mas pula `extracao`/`item_extraido` (é publicado diretamente, já que é humano).

### 4.8 Feed ICS

```sql
create table ics_token (
  id           uuid primary key default gen_random_uuid(),
  membro_id    uuid not null references membro(id) on delete cascade,
  token_hash   text not null unique,       -- sha256 do token; o token em claro só é mostrado uma vez
  escopo       text not null check (escopo in ('meus','casal')),
  criado_em    timestamptz not null default now(),
  revogado_em  timestamptz
);
```

### 4.9 Diagrama de relações (resumo)

```
colecao 1─N programa 1─N temporada 1─N sessao
                   │ 1─N sessao (avulsa)
                   │ 1─N janela 1─N janela_horario
                   │              1─N janela_excecao
                   │ 1─N escolha ──→ sessao | janela(+data)
                   │ N─N fonte (programa_fonte)
local 1─N sessao / temporada / janela
fonte 1─N extracao 1─N item_extraido ──→ programa (após aprovação)
membro 1─N escolha (marcado_por), 1─N ics_token
```

### 4.10 Volume esperado

Mesmo ingerindo uma mostra grande (≈ 400 filmes × 3 sessões) e um ano de teatro e música, ficamos na casa de dezenas de milhares de sessões — trivial para Postgres com o índice em `sessao.inicio`. O maior consumidor de espaço será `fonte.texto_bruto`; ainda assim muito abaixo dos 500 MB do plano gratuito.

---

## 5. Consultas dos calendários

### 5.1 Expansão unificada de ocorrências num período

Função SQL que devolve, para um intervalo de dias, **todas** as ocorrências — sessões reais e dias virtuais das janelas — num formato comum:

```sql
create or replace function ocorrencias_periodo(p_inicio date, p_fim date)
returns table (
  origem       text,          -- 'sessao' | 'janela'
  programa_id  uuid,
  local_id     uuid,
  sessao_id    uuid,
  janela_id    uuid,
  dia          date,
  inicio       timestamptz,
  fim          timestamptz,
  gratuito     boolean
)
language sql stable as $$
  -- (a) Sessões pontuais (inclui as geradas por temporadas)
  select 'sessao', s.programa_id, s.local_id, s.id, null::uuid,
         (s.inicio at time zone 'America/Sao_Paulo')::date,
         s.inicio, s.fim, s.gratuito
  from sessao s
  where not s.cancelada
    and s.inicio >= (p_inicio::timestamp       at time zone 'America/Sao_Paulo')
    and s.inicio <  ((p_fim + 1)::timestamp    at time zone 'America/Sao_Paulo')

  union all

  -- (b) Janelas: um "dia virtual" por dia aplicável, somente dentro do intervalo pedido
  select 'janela', j.programa_id, j.local_id, null::uuid, j.id, d.dia,
         (d.dia + h.abre)  at time zone 'America/Sao_Paulo',
         (d.dia + h.fecha) at time zone 'America/Sao_Paulo',
         j.gratuito
  from janela j
  cross join lateral (
    select g::date as dia
    from generate_series(greatest(j.data_inicio, p_inicio),
                         least(j.data_fim, p_fim), interval '1 day') g
  ) d
  join janela_horario h
    on h.janela_id = j.id and h.dia_semana = extract(isodow from d.dia)
  left join janela_excecao e
    on e.janela_id = j.id and e.data = d.dia
  where daterange(j.data_inicio, j.data_fim, '[]') && daterange(p_inicio, p_fim, '[]')
    and e.janela_id is null                      -- dias sem exceção

  union all

  -- (c) Janelas: dias com exceção de abertura (horário especial ou abertura extraordinária)
  select 'janela', j.programa_id, j.local_id, null::uuid, j.id, e.data,
         (e.data + e.abre)  at time zone 'America/Sao_Paulo',
         (e.data + e.fecha) at time zone 'America/Sao_Paulo',
         j.gratuito
  from janela j
  join janela_excecao e on e.janela_id = j.id
  where not e.fechado and e.data between p_inicio and p_fim
$$;
```

Custo: o `generate_series` só percorre a interseção entre a janela e o período visível (tipicamente uma semana ou um mês). Com 200 exposições em cartaz e um mês na tela, são ~6 mil linhas virtuais, calculadas em milissegundos e **nunca gravadas**.

### 5.2 Calendário "Cardápio de Possibilidades"

```sql
-- Parâmetros de filtro opcionais: p_tipos, p_bairros, p_locais, p_gratuito
select o.*, p.titulo, p.tipo, l.nome as local_nome, l.bairro,
       coalesce(o.gratuito, p.gratuito) as eh_gratuito,
       es.status as status_sessao,                 -- p/ marcar visualmente o que já foi escolhido
       ep.status as status_programa
from ocorrencias_periodo(:inicio, :fim) o
join programa p on p.id = o.programa_id and p.mesclado_em is null
join local    l on l.id = o.local_id
left join escolha ep on ep.programa_id = p.id and ep.sessao_id is null and ep.janela_id is null
left join escolha es on es.sessao_id = o.sessao_id
where coalesce(ep.status, 'interesse') <> 'descartado'     -- programa descartado some
  and coalesce(es.status, 'agendado')  <> 'descartado'     -- sessão descartada some
  and (:tipos    is null or p.tipo   = any(:tipos))
  and (:bairros  is null or l.bairro = any(:bairros))
  and (:locais   is null or l.id     = any(:locais))
  and (:gratuito is null or coalesce(o.gratuito, p.gratuito) = :gratuito)
order by o.dia, (o.origem = 'janela') desc, o.inicio;
```

Na prática isso vira uma função `cardapio_possibilidades(p_inicio, p_fim, p_filtros jsonb)` chamada via RPC do Supabase.

**Apresentação (importante para não poluir):** em cada dia, as janelas aparecem agrupadas numa faixa recolhível "Em cartaz hoje (N)", separada das sessões com horário. Sem isso, uma exposição de 45 dias ocuparia todos os dias da grade e esconderia o que é realmente pontual.

### 5.3 Calendário "Encontro Marcado"

View que normaliza sessões e visitas em "compromissos" com início/fim concretos:

```sql
create or replace view compromisso as
select e.id as escolha_id, e.status, e.participantes, e.valor_pago, e.codigo_ingresso,
       e.observacoes, e.programa_id,
       coalesce(s.local_id, j.local_id) as local_id,
       coalesce(s.inicio,
                ((e.data_visita + coalesce(e.hora_visita, time '14:00'))
                   at time zone 'America/Sao_Paulo')) as inicio,
       coalesce(s.fim,
                s.inicio + make_interval(mins => coalesce(p.duracao_min, 120)),
                ((e.data_visita + coalesce(e.hora_visita, time '14:00'))
                   at time zone 'America/Sao_Paulo') + interval '2 hours') as fim,
       (e.janela_id is not null) as eh_visita,
       coalesce(s.cancelada, false) as cancelada
from escolha e
join programa p on p.id = e.programa_id
left join sessao s on s.id = e.sessao_id
left join janela j on j.id = e.janela_id
where e.status in ('agendado','ingresso_comprado')
  and (e.sessao_id is not null or e.janela_id is not null);
```

Consulta do calendário: `select … from compromisso where inicio >= :ini and inicio < :fim [and :membro = any(participantes)] order by inicio`.

Distinção visual: **compromisso em aberto** (`agendado`) = contorno tracejado/cor clara; **ingresso comprado** = bloco cheio com ícone de ingresso. Sessões canceladas pela organização aparecem riscadas com alerta. Sobreposição de horários entre compromissos gera aviso.

> A view deve ser criada com `security_invoker = true` para respeitar a RLS das tabelas subjacentes.

### 5.4 Cardápio Cultural (lista)

Mesma base da §5.2, mas agrupada por **programa** (cartão com título, tipo, local(is), próximas sessões ou "em cartaz até dd/mm") em vez de por dia. Filtros: período, local, bairro, tipo, gratuito/pago, coleção, e "mostrar descartados". Ações no cartão:

- **Quero ir** → se houver 1 ocorrência, agenda direto; se houver várias, abre seletor de sessões (multisseleção); se for janela, pede data (e hora opcional) da visita. Pergunta "quem vai" (padrão: ambos).
- **Não tenho interesse** → `descartado` no nível do programa, com "Desfazer" imediato (toast) e recuperação na aba Descartados.
- **Ingresso comprado** → promove `agendado` para `ingresso_comprado` e abre campos opcionais (valor, código, anexo).

---

## 6. Ingestão e extração assistida por LLM

### 6.1 Pipeline

```
Entrada (URL | PDF | texto | [imagem]) 
  → fonte (snapshot + hash; se hash já existe, avisa "já importado")
  → pré-processamento determinístico
       URL: fetch → JSON-LD schema.org/Event (se houver) → Readability → texto limpo
       PDF: upload direto ao Storage → fatiamento por páginas
       Texto: como veio
  → extracao (1..N partes) → chamada ao LLM com structured output
  → validação (Zod) + verificação de trechos (o trecho citado existe no texto da fonte?)
  → casamento de local (alias/trigram) e candidatos a duplicata
  → item_extraido (rascunho) → FILA DE REVISÃO
  → humano edita/aprova/rejeita/mescla → programa/sessões/janelas/temporadas + proveniência
```

### 6.2 Schema de extração (resumo)

Um item por programa encontrado:

```jsonc
{
  "programa": { "titulo": "...", "titulo_original": null, "tipo": "teatro", "subtipo": null,
                "descricao": "...", "ficha": {...}, "duracao_min": 90, "classificacao": "14",
                "gratuito": false, "preco_min": 40, "preco_max": 80, "preco_obs": "...",
                "links": [...], "colecao": "Mostra X" },
  "local": { "nome": "...", "endereco": "...", "bairro": "..." },
  "sessoes":    [ { "inicio": "2026-11-15T20:00", "sala": null, "versao": null, "link_ingresso": null } ],
  "temporadas": [ { "data_inicio": "2026-11-06", "data_fim": "2026-12-14",
                    "regra": [{"dias":[4,5,6],"hora":"20:00"},{"dias":[7],"hora":"18:00"}],
                    "datas_excluidas": [] } ],
  "janelas":    [ { "data_inicio": "2026-11-01", "data_fim": "2026-12-15",
                    "horarios": [{"dias":[2,3,4,5,6,7],"abre":"10:00","fecha":"18:00"}] } ],
  "trechos":    [ { "campo": "temporadas[0]", "texto": "<citação literal da fonte>" } ],
  "alertas":    [ "ano não informado; inferido 2026" ],
  "confianca":  0.0–1.0
}
```

Regras do prompt: datas/hora sempre locais de São Paulo (sem fuso no JSON); a data de referência (data da coleta) é informada para inferir ano; **toda data/hora deve vir acompanhada de um trecho literal** da fonte; preferir `temporada` quando o texto descreve recorrência e `janela` para exposições/visitação; nunca inventar campos ausentes (usar `null`).

**Verificação anti-alucinação:** cada `trecho` é procurado (busca normalizada/aproximada) no `texto_bruto` da fonte; se não for encontrado, o item recebe alerta e é destacado na revisão. (A API não permite combinar *citations* nativas com *structured outputs*, por isso a verificação é nossa.)

### 6.3 Particularidades por tipo de entrada

| Entrada | Estratégia | Limitações |
|---|---|---|
| **URL** | `fetch` no servidor → JSON-LD `Event` (mapeado sem LLM quando completo) → Readability → texto para o LLM. | Páginas renderizadas por JavaScript, *paywalls* e proteção anti-bot (Cloudflare etc.) podem devolver HTML vazio ou bloqueado. Plano B na UI: "não consegui ler — cole o texto da página". |
| **PDF** | Upload direto do navegador ao Storage (URL assinada). Até ~15–20 páginas por parte; envio como documento PDF nativo (o modelo lê texto e layout). Para PDFs só-texto muito grandes, alternativa mais barata: extrair texto com `pdfjs` e enviar só o texto. | Catálogos com grade visual (tabela de horários) dependem da leitura de layout; PDFs escaneados exigem leitura de imagem (mais tokens). |
| **Texto colado** | Direto ao LLM. | Nenhuma relevante. |
| **Manual** | Formulário com os mesmos campos; publica direto. | — |
| **[Sugestão] Imagem/print** | Envio da imagem ao LLM (visão). | Muita programação de SP circula só no Instagram, que não é raspável; um print resolve. **[DECISÃO]** se entra no escopo. |

### 6.4 Execução assíncrona

- Cada `extracao` (parte) é processada por uma invocação independente de Route Handler, com *streaming* da resposta da API para não estourar tempo de requisição.
- A UI mostra o progresso (parte 3/8) consultando a tabela `extracao`; falhas ficam com `status = 'erro'` e botão "tentar de novo".
- Limites de segurança por parte: máximo de páginas, `max_tokens` definido, e teto mensal de gasto configurado também no Console da Anthropic.

### 6.5 Modelo de LLM e custo estimado

Preços de tabela da API (USD por milhão de tokens, consultados em out/2026): Claude Opus 5.5 — US$ 4 entrada / US$ 20 saída; Claude Sonnet 5.5 — US$ 2 / US$ 10; Claude Haiku 5.5 — US$ 0,10 / US$ 0,50 (até 100 mil tokens de entrada por requisição). A Batch API custa 50% disso, mas pode levar horas — inadequada para o fluxo interativo de revisão.

Premissas por unidade (incluindo tokens de raciocínio na saída e ~3 mil tokens de instruções+schema por chamada):

| Unidade | Entrada | Saída | Opus 5.5 | Sonnet 5.5 | Haiku 5.5 |
|---|---|---|---|---|---|
| 1 página web | ~18 mil | ~5 mil | US$ 0,17 | US$ 0,09 | US$ 0,004 |
| 1 catálogo PDF de 40 p. | ~80 mil | ~25 mil | US$ 0,82 | US$ 0,41 | US$ 0,02 |
| 1 texto colado | ~5 mil | ~2 mil | US$ 0,06 | US$ 0,03 | US$ 0,002 |

Cenários mensais:

| Cenário | Volume | Opus 5.5 | Sonnet 5.5 | Haiku 5.5 |
|---|---|---|---|---|
| Leve | 20 URLs, 1 PDF, 15 textos | ~US$ 5 | ~US$ 2,6 | ~US$ 0,15 |
| Moderado | 60 URLs, 4 PDFs, 30 textos | ~US$ 15 | ~US$ 8 | ~US$ 0,40 |
| Intenso (mês de mostra grande) | Moderado + catálogo de ~300 p. | ~US$ 22 | ~US$ 11 | ~US$ 0,55 |

São estimativas de ordem de grandeza; o custo real por extração será registrado (`extracao.custo_usd`) e exibido numa tela de uso a partir da Fase 5. **Recomendação:** o modelo fica configurável por variável de ambiente; na Fase 5, montar um pequeno conjunto de referência (10–15 fontes reais conferidas à mão) e comparar a qualidade dos três modelos antes de fixar o padrão. O padrão provisório é Opus 5.5 com esforço baixo/médio. **[DECISÃO]**

Infraestrutura (Vercel Hobby + Supabase Free): US$ 0. Domínio próprio opcional (~R$ 40/ano para `.com.br`).

---

## 7. Autenticação, privacidade e segurança

- Supabase Auth com **cadastro desabilitado**; as duas contas são criadas manualmente no painel e inseridas em `membro`. Método de login **[DECISÃO]**: e-mail+senha (simples, funciona bem como PWA), *magic link* (sem senha, mas no celular abre o navegador em vez do app instalado) ou Google OAuth (um toque, se ambos usam Google).
- Middleware do Next.js bloqueia todas as rotas para não autenticados; RLS bloqueia no banco mesmo que a aplicação falhe.
- `ANTHROPIC_API_KEY` e `SUPABASE_SERVICE_ROLE_KEY` só no servidor (nunca com prefixo `NEXT_PUBLIC_`).
- **Feed ICS** precisa ser acessível sem login (aplicativos de calendário não fazem login): URL com token aleatório de 32 bytes (`/api/ics/<token>.ics`); guarda-se apenas o hash; pode ser revogado e regenerado. Quem tiver a URL vê o *Encontro Marcado* daquele escopo — risco aceitável se a URL não for compartilhada.
- Conteúdo raspado é **dado não confiável**: o LLM só recebe a tarefa de extrair para um schema fixo, sem ferramentas, e tudo passa pela revisão humana — isso neutraliza tentativas de *prompt injection* embutidas em páginas.
- `robots.txt` e frequência baixa de coleta respeitados; uso estritamente pessoal.

### 7.1 Feed ICS — detalhes

- `VCALENDAR` com `X-WR-TIMEZONE:America/Sao_Paulo` e `VTIMEZONE`; `DTSTART;TZID=America/Sao_Paulo`.
- `UID` estável = `<escolha.id>@agenda-rolezin` (alterações atualizam o evento em vez de duplicar).
- Título: `🎟️ Título` para ingresso comprado, `Título (a confirmar)` para agendado; `STATUS:CONFIRMED`/`TENTATIVE`; `STATUS:CANCELLED` para sessão cancelada.
- `LOCATION` com nome + endereço; `DESCRIPTION` com quem vai, valor, código, link.
- Escopos: `meus` (só onde o membro está em `participantes`) ou `casal` (tudo).
- **Expectativa realista:** o Google Agenda atualiza calendários assinados com pouca frequência (pode levar muitas horas); o Apple Calendar permite configurar o intervalo. Não serve para mudanças de última hora.

---

## 8. Fases de implementação e critérios de aceite

Cada fase termina com deploy em produção, testes passando e demonstração no celular.

### Fase 0 — Fundação
- Repositório Next.js + TS + Tailwind + shadcn/ui; ESLint/Prettier; Vitest; Playwright.
- Projeto Supabase; migração inicial com extensões, `membro`, `eh_membro()`; RLS.
- Auth com as duas contas; middleware de proteção; layout mobile-first em pt-BR.
- Deploy na Vercel; variáveis de ambiente; *keepalive* (se aprovado).

**Aceite:** (1) os dois conseguem entrar no celular e no desktop; (2) um terceiro e-mail não consegue se cadastrar nem acessar nada; (3) requisição anônima à API REST do Supabase devolve vazio para qualquer tabela; (4) `pnpm lint`, `pnpm typecheck` e `pnpm test` passam; (5) limites vigentes de Vercel/Supabase verificados e anotados no CLAUDE.md.

### Fase 1 — Modelo de dados e cadastro manual
- Migrações de §4 (exceto ICS); tipos gerados.
- CRUD de local (com aliases), coleção, programa, sessão avulsa, temporada (com expansão) e janela (horários + exceções).
- Função pura de expansão de temporada + testes; reexpansão segura (§4.5.1).

**Aceite:** (1) cadastrar uma peça "qui–sáb 20h, dom 18h, 06/11–14/12, exceto 20/11" gera exatamente as sessões esperadas (teste unitário com contagem e datas); (2) alterar a regra preserva sessões com escolha; (3) cadastrar exposição "01/11–15/12, ter–dom 10–18h, fechada 25/12" não cria nenhuma linha por dia; (4) horários exibidos e gravados corretos em `America/Sao_Paulo` (teste com instante UTC conhecido); (5) formulários utilizáveis em tela de 375 px.

### Fase 2 — Cardápio Cultural e escolhas
- `ocorrencias_periodo` e `cardapio_possibilidades` (SQL) + testes.
- Lista com filtros (período, local, bairro, tipo, gratuito/pago, coleção); filtros na URL (compartilháveis entre os dois).
- Ações: Quero ir (seletor de sessões / data de visita; quem vai), Não tenho interesse (com desfazer), Ingresso comprado (valor, código, anexo); aba Descartados; histórico.

**Aceite:** (1) cada filtro, isolado e combinado, devolve o conjunto esperado num *fixture* de teste; (2) descartar some do cardápio e reaparece ao desfazer; (3) "Quero ir" num programa com 5 sessões permite escolher 2; (4) "quem vai" exibido corretamente para cada um dos dois usuários ("Eu"/nome do outro); (5) `escolha_historico` registra quem/quando em toda alteração.

### Fase 3 — Calendários
- Visão "agenda" (lista por dia) como padrão no celular; semana/mês no desktop; alternância Possibilidades ↔ Encontro Marcado persistida por usuário.
- Faixa recolhível "Em cartaz hoje" para janelas; estilos distintos agendado × ingresso comprado; alerta de sobreposição e de sessão cancelada.

**Aceite:** (1) Possibilidades nunca mostra descartados; (2) Encontro Marcado mostra só `agendado`/`ingresso_comprado`, com distinção visual verificável por teste e2e; (3) visita a exposição aparece no dia/hora escolhidos; (4) carregar um mês com ≥ 2.000 ocorrências responde em < 1 s no servidor; (5) navegação e ações utilizáveis com uma mão no celular (teste e2e em viewport móvel).

### Fase 4 — Exportação ICS
- Tabela `ics_token`, tela para gerar/revogar URL, endpoint `.ics`.

**Aceite:** (1) o arquivo valida em um validador iCalendar; (2) assinado no Google Agenda e no Apple Calendar, os eventos aparecem no horário correto; (3) mudar status/horário atualiza o mesmo evento (UID estável) após a atualização do cliente; (4) token revogado devolve 404; (5) escopo `meus` não inclui compromissos só do outro.

### Fase 5 — Extração por LLM (texto colado) e fila de revisão
- `fonte`, `extracao`, `item_extraido`, `programa_fonte`; cliente Anthropic; schema Zod; prompt; verificação de trechos; casamento de local.
- Fila de revisão: lista de rascunhos, edição em formulário (mesmos componentes da Fase 1), aprovar/rejeitar, ver trecho de origem ao lado de cada campo.
- Registro de tokens e custo; tela de uso mensal.
- Conjunto de referência (10–15 fontes) e comparação de modelos.
- Sugestão simples de duplicata por similaridade de título (a mesclagem completa vem na Fase 7).

**Aceite:** (1) colar o texto de uma programação com sessões, temporada e exposição produz rascunhos nos três tipos corretos; (2) nada aparece no Cardápio antes de aprovado; (3) trechos não encontrados na fonte geram alerta visível; (4) rejeitar mantém o item para auditoria; (5) cada extração registra modelo, tokens e custo; (6) no conjunto de referência, ≥ 90% das datas/horas corretas sem edição (meta a confirmar após a medição).

### Fase 6 — URLs e PDFs
- Coleta de URL com JSON-LD → Readability → LLM; detecção de página vazia/bloqueada com fallback para "cole o texto".
- Upload de PDF ao Storage, fatiamento, processamento por partes com progresso e retomada.
- Detecção de reenvio pela `conteudo_hash`.

**Aceite:** (1) testado em ≥ 5 sites reais escolhidos pelo casal, com resultado documentado (funciona / funciona parcialmente / bloqueado); (2) um catálogo PDF de ≥ 40 páginas é processado inteiro, com falha de uma parte recuperável sem refazer as demais; (3) reimportar a mesma URL sem mudança avisa em vez de duplicar.

### Fase 7 — Deduplicação e mesclagem
- Candidatos por: título normalizado (trigram ≥ limiar), mesmo tipo, sobreposição de período e/ou mesmo local; `nao_duplicata` respeitado.
- Na revisão: "mesclar com existente" (escolha campo a campo; sessões/janelas novas anexadas; sessões idênticas deduplicadas por `programa+local+início`).
- Mesclagem de dois programas já aprovados: move sessões, janelas, temporadas, escolhas e fontes para o destino numa transação; origem fica com `mesclado_em`.

**Aceite:** (1) o mesmo filme vindo do site do cinema e do catálogo da mostra é sugerido como duplicata; (2) mesclar preserva todas as fontes na proveniência e todas as escolhas; (3) marcar "não é duplicata" impede nova sugestão do par; (4) nenhuma mesclagem ocorre sem confirmação humana.

### Fase 8 — Refinamentos (priorizar depois)
- PWA instalável (manifest, ícones, cache offline de leitura do Encontro Marcado).
- Monitoramento de fontes (`monitorar = true`): recoleta periódica, só reextrai se o hash mudar.
- Entrada por imagem/print (se aprovada); mapa por bairro; "fomos/avaliação" pós-evento.

**Aceite:** definido quando a fase for priorizada.

---

## 9. Riscos e decisões que dependem de vocês

### 9.1 Riscos

| # | Risco | Impacto | Mitigação |
|---|---|---|---|
| R1 | **Raspagem frágil**: sites renderizados por JS, anti-bot, mudanças de layout; Instagram não é raspável | Alto — é a principal fonte de atrito | JSON-LD primeiro; fallback "cole o texto"; entrada por print (sugestão); não prometer coleta automática universal |
| R2 | **Datas ambíguas** ("sex 14", sem mês/ano; "últimas semanas") e alucinação do LLM | Alto — compromisso no dia errado | Revisão obrigatória; trecho literal por data; verificação do trecho na fonte; alertas de ano inferido |
| R3 | Pausa do Supabase gratuito por inatividade | Médio | Keepalive diário, ou plano pago, ou Neon |
| R4 | Limites de duração/tamanho das funções da Vercel Hobby | Médio — PDFs grandes | Upload direto ao Storage; fatiamento; streaming; estado persistido |
| R5 | Custo de LLM fugir do controle (ex.: PDF de 300 páginas reprocessado várias vezes) | Baixo/Médio | Teto mensal no Console; hash anti-reenvio; custo registrado por extração; confirmação antes de processar PDFs grandes |
| R6 | Atualização lenta de feeds ICS no Google Agenda | Médio — expectativa | Documentar; o app é a fonte de verdade para mudanças de última hora |
| R7 | Vazamento da URL do ICS | Baixo | Token revogável; só hash no banco |
| R8 | Fuso horário (eventual retorno do horário de verão no Brasil) | Baixo | Sempre `timestamptz` + conversão por `America/Sao_Paulo`, nunca offset fixo `-03:00` |
| R9 | Regenerar temporada apagar sessão já escolhida | Médio | Reexpansão segura (§4.5.1) + teste |
| R10 | Excesso de escopo para um projeto pessoal (manutenção) | Médio | Fases curtas; Fase 8 só se as anteriores estiverem em uso real |

### 9.2 Decisões **[DECISÃO]**

1. **Modelo de LLM:** manter Opus 5.5 como padrão provisório e decidir após medir na Fase 5 (recomendado), ou fixar já Sonnet 5.5 / Haiku 5.5 por custo?
2. **Login:** e-mail+senha, magic link ou Google OAuth?
3. **Pausa do Supabase:** keepalive automático (recomendado), plano Pro (US$ 25/mês) ou trocar por Neon?
4. **Semântica dos status:** confirma que `interesse` = "talvez" no nível do programa (sem sessão escolhida), e "Quero ir" = `agendado` numa sessão/visita? E que "Não tenho interesse" descarta o **programa** inteiro (com opção secundária de descartar só uma sessão)?
5. **Quem vai:** padrão "ambos"? É necessário registrar idas separadas (cada um numa sessão diferente do mesmo programa)? O modelo já suporta.
6. **Visita a exposição:** hora opcional com padrão 14h e duração 2h no calendário — ok?
7. **ICS:** um feed por pessoa (`meus`), um do casal, ou ambos?
8. **Entrada por imagem/print** (para Instagram): incluir no escopo (Fase 6) ou deixar para a Fase 8?
9. **Fontes prioritárias:** listar as 5–10 páginas que vocês mais usam (ex.: centros culturais, cinemas, bilheterias) para testar a coleta já na Fase 6 e calibrar expectativas.
10. **Bairro/região:** texto livre com autocomplete basta, ou querem lista controlada (ex.: distritos oficiais + regiões)?
11. **Domínio:** subdomínio gratuito `*.vercel.app` ou domínio próprio?
12. **Retenção:** guardar indefinidamente rascunhos rejeitados e o texto bruto das fontes (auditoria) ou limpar após N meses?

---

## 10. Glossário de UI (pt-BR)

| Termo na UI | Significado técnico |
|---|---|
| Cardápio Cultural | Lista filtrável de programas (§5.4) |
| Cardápio de Possibilidades | Calendário de todas as ocorrências não descartadas (§5.2) |
| Encontro Marcado | Calendário de compromissos (§5.3) |
| Quero ir | Criar escolha `agendado` em sessão/visita |
| Não tenho interesse | Escolha `descartado` no programa |
| Ingresso comprado | Escolha `ingresso_comprado` |
| Em cartaz | Janela contínua ativa no dia |
| Revisar | Fila de `item_extraido` em rascunho |
