# Vezulo — Sistema de Métricas e Reputação

**Especificação do produto para implementação em WordPress headless**
Versão 1.0 · Julho/2026 · Base: análise do Indeed Brasil + telas atuais do Vezulo

---

## 1. Contexto e objetivo

O Vezulo hoje tem as telas desenhadas (13 do candidato, 10 da empresa), o backend headless em WordPress já pronto para autenticação, perfis, vagas e candidaturas, mas **os números que aparecem nos dashboards ainda são de exemplo** (conforme o progresso da Fase 1). Este documento define, métrica por métrica, *o que cada número significa, como é calculado, de onde vêm os dados e como implementar no WordPress*, para que a Fase 2 saia do placeholder e vire produto real.

A referência escolhida foi o **Indeed Brasil**, líder de mercado. Estudamos o funcionamento dos dois lados (candidato e empregador), extraímos o que funciona, apontamos o que o Indeed faz mal ou não faz, e desenhamos um sistema que aproveita o diferencial do Vezulo: **match por IA explicável + economia de créditos (Vezus)**.

---

## 2. Como o Indeed funciona (resumo da análise)

### 2.1 Lado do candidato no Indeed

O perfil do candidato só fica visível para empresas quando ele ativa a opção "as empresas podem encontrar o seu perfil". Ativado, o Indeed compartilha experiência profissional, formação, competências, certificações e cidade/estado. O nome só é revelado depois que o candidato responde a uma mensagem da empresa, e o e-mail é mascarado por um alias — privacidade por padrão.

O sinal mais importante e pouco óbvio: o Indeed mostra às empresas **quando o candidato editou o perfil pela última vez e quando visitou o Indeed pela última vez**. Ou seja, *atividade e recência* são parte da métrica — um perfil "vivo" vale mais que um perfil parado. O candidato também recebe status de retorno das candidaturas (incluindo o polêmico "não selecionado pelo empregador"), que gera frustração porque é opaco: não diz o porquê.

### 2.2 Lado da empresa no Indeed

O empregador acompanha cada vaga por um funil de desempenho e um funil de candidatos:

- **Desempenho da vaga:** impressões → cliques → candidaturas, com CPC (custo por clique), gasto total, taxa de candidaturas e origem das candidaturas (patrocinada padrão vs. premium).
- **Funil de candidatos (6 estágios):** Novo/Aguardando avaliação → Avaliado (Sim/Talvez/Não) → Contatado → Entrevista → Contratado → Recusado. Há filtros por localização, status, respostas às perguntas de triagem e data. Perguntas marcadas como "indispensáveis" recusam automaticamente quem não atende.
- **Candidatos correspondentes (matched candidates):** ao patrocinar a vaga, o Indeed sugere candidatos cujo perfil combina com os requisitos e permite convidá-los.
- **Perfil e reputação da empresa:** páginas de empresa com avaliações de funcionários, nota em estrelas, salários e sinais de confiança.

### 2.3 O que dá para melhorar em cima do Indeed

| Ponto do Indeed | Limitação | Oportunidade no Vezulo |
|---|---|---|
| Match/relevância | Caixa-preta: o candidato não sabe por que "não foi selecionado" | **Match explicável**: mostrar o *porquê* do % (skills, senioridade, local, salário) dos dois lados |
| Atividade do perfil | Só mostra "visto por último" | Transformar em **status de disponibilidade** (Disponível / Aberto a propostas / Empregado) e usar como peso no match |
| Reputação da empresa | Avaliações existem, mas soltas do fluxo de contratação | **Taxa de resposta** e **avaliação da empresa** dentro do fluxo, protegendo o candidato que "gasta" tempo |
| Custo | CPC/gasto por clique, difícil prever ROI | **Vezus por desbloqueio** + métricas de **ROI (Vezus por contratação)** claras |
| Triagem | Perguntas indispensáveis | Manter, e somar **score de compatibilidade** para ordenar, não só filtrar |
| Qualidade do gasto | Empresa pode pagar e receber lixo | Como a empresa **paga por desbloqueio**, o Vezulo precisa de **sinais de qualidade** (perfil verificado, completo, ativo) antes de cobrar o Vezu |

O último ponto é estratégico: no modelo do Vezulo a empresa gasta um Vezu para desbloquear um perfil. Se o perfil for fraco/desatualizado, ela sente que "queimou" crédito e para de comprar. **Portanto todo o sistema de métricas do candidato existe para proteger a economia de Vezus.**

---

## 3. Arquitetura do sistema de métricas

Cinco motores independentes, todos reaproveitando o backend já existente (CPTs `vz_experience`, `vz_education`, `vz_certification`, `vz_vaga`, `vz_candidatura`; roles `vezulo_candidato`/`vezulo_empresa`):

1. **Completude & Qualidade do Perfil** (candidato e empresa)
2. **Match Score IA explicável** (candidato ↔ vaga ↔ empresa)
3. **Rastreamento de Eventos** (visualizações, interesses, desbloqueios, mensagens) → alimenta todos os contadores dos dashboards
4. **Reputação & Confiança** (verificação, taxa de resposta, avaliação, atividade)
5. **Economia de Vezus** (saldo, consumo, ROI, ledger de transações)

E dois derivados de apresentação: **Gamificação/Conquistas** (candidato) e **KPIs de Dashboard** (os dois lados).

---

## 4. Motor 1 — Completude e qualidade do perfil

### 4.1 Candidato — "Perfil Completo %"

Aparece no Dashboard (92% na tela) e alimenta "Próximos Passos". É uma soma ponderada de itens preenchidos. Peso proporcional ao impacto do item no match e na decisão do recrutador:

| Item | Peso | Condição para pontuar |
|---|---:|---|
| Foto de perfil | 5% | avatar enviado |
| Dados básicos | 10% | nome + cargo desejado + cidade/estado + telefone |
| Resumo profissional | 10% | texto ≥ 200 caracteres |
| Experiência | 20% | ≥ 1 registro (`vz_experience`) com descrição |
| Formação | 10% | ≥ 1 registro (`vz_education`) |
| Hard skills | 15% | ≥ 5 skills com nível definido |
| Soft skills | 5% | ≥ 3 tags |
| Certificações | 10% | ≥ 1 registro (`vz_certification`) |
| Portfólio / links | 5% | LinkedIn ou link de portfólio válido |
| Teste de skills | 10% | ao menos 1 teste concluído |
| **Total** | **100%** | |

**Fórmula:** `completude = Σ (peso_item × item_preenchido)`, arredondado ao inteiro.

**Próximos Passos** = os 2–3 itens não pontuados de **maior peso**, ordenados por peso decrescente, com o ganho exibido ("Adicionar portfólio → +5%"; "Fazer teste de skills → +10%"). A tela atual já mostra "Adicionar portfólio (aumenta suas chances em 40%)" — recomendo trocar o texto genérico pelo ganho real de completude e/ou pelo ganho médio de match observado.

### 4.2 Empresa — "Perfil da Empresa completo/verificado"

A empresa não tem barra de % na tela, mas tem o selo **Verificada**. Proposta de checklist de verificação (booleano por item; selo "Verificada" exige CNPJ + e-mail corporativo + ≥ 4 itens):

CNPJ validado (Receita/regex + dígito verificador), e-mail de domínio corporativo, logo, razão social, segmento, tamanho, website ativo, LinkedIn. Empresa verificada aparece com mais destaque na busca do candidato e ganha peso de confiança (ver Motor 4).

---

## 5. Motor 2 — Match Score IA (explicável)

O número verde de "MATCH" que aparece em Vagas Recomendadas (95/89/82%), em Buscar Talentos (97/92%) e em Candidatos Favoritos. É **o mesmo motor** rodando nas duas direções.

### 5.1 Fórmula

```
Match(candidato, vaga) = 100 × Σ ( peso_dimensão × score_dimensão )
```

com cada `score_dimensão` normalizado entre 0 e 1:

| Dimensão | Peso | Como pontua (0–1) |
|---|---:|---|
| Hard skills | 40% | interseção ponderada entre skills exigidas (com nível) e skills do candidato. `Σ min(nível_cand, nível_exig) / Σ nível_exig` |
| Senioridade / experiência | 20% | proximidade entre anos de experiência e cargo exigido vs. do candidato (1 = igual, cai com a distância) |
| Localização / modalidade | 15% | remoto = 1; híbrido/presencial = 1 se mesma cidade/região, decai com a distância |
| Formação / certificações | 10% | requisitos atendidos ÷ requisitos exigidos |
| Pretensão salarial | 10% | 1 se dentro da faixa; decai fora dela; "a combinar" = neutro (0,7) |
| Atividade / recência | 5% | candidato ativo nos últimos 7 dias = 1; decai até 0 em 90 dias |

Pesos ficam em `option` editável (`vezulo_match_weights`) para calibrar sem deploy. A soma dos pesos deve ser 1.

### 5.2 Explicabilidade (o diferencial vs. Indeed)

Guardar o vetor de scores por dimensão junto com o total. Na UI, ao passar o mouse/expandir o card de match, mostrar as barras por dimensão ("Skills 95% · Senioridade 80% · Local 100% · Salário 70%"). Isso resolve a dor do "não selecionado" opaco do Indeed e ajuda o candidato a melhorar o próprio perfil (fecha o ciclo com o Motor 1).

### 5.3 Cálculo e cache

- **Candidato → vagas recomendadas:** calcular sob demanda contra vagas Ativas, cachear por 24 h em `wp_vezulo_match` (`candidate_id`, `vaga_id`, `score`, `breakdown_json`, `computed_at`). Recalcular quando o perfil muda (hook em salvar skills/experiência) ou quando uma vaga nova é publicada.
- **Empresa → Buscar Talentos / Match Inteligente:** roda a busca (filtros + prompt em linguagem natural) e ordena por score. O prompt de linguagem natural ("Procuro um farmacêutico em Curitiba com experiência em manipulação") vira filtros + boost semântico.
- **Custo IA:** a geração de ranking (Match Inteligente) e a busca podem ser gratuitas; **cobra-se Vezu só no desbloqueio** do perfil (ver Motor 5). Isso mantém o topo do funil grátis e monetiza a intenção real.

---

## 6. Motor 3 — Rastreamento de eventos

Praticamente todos os contadores dos dashboards são **agregações de eventos**. Um único log resolve tudo.

### 6.1 Tabela de eventos

Tabela custom `wp_vezulo_events` (volume alto → não usar CPT/postmeta):

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | BIGINT PK | |
| `type` | VARCHAR | `profile_view`, `vaga_view`, `vaga_apply`, `interest`, `unlock`, `message`, `match_generated`, `hire` |
| `actor_id` | BIGINT | quem gerou (empresa ou candidato) |
| `target_id` | BIGINT | alvo (candidato, vaga, empresa) |
| `vaga_id` | BIGINT NULL | contexto de vaga, quando houver |
| `meta_json` | TEXT NULL | dados extras (ex.: vezus gastos) |
| `created_at` | DATETIME | índice |

Índices em (`type`,`target_id`,`created_at`) e (`type`,`actor_id`,`created_at`).

### 6.2 De onde sai cada número da tela

**Candidato:**

| Métrica na tela | Consulta |
|---|---|
| Visualizações (209) | count `profile_view` where `target_id`=candidato |
| Empresas interessadas (14) | count distinct `actor_id` em `interest` where `target_id`=candidato |
| Matches realizados (8) | count matches ≥ limiar (ex.: ≥ 80%) que viraram interação mútua |
| Evolução do Perfil (7 dias) | `profile_view` agrupado por dia, últimos 7 dias |

**Empresa:**

| Métrica na tela | Consulta |
|---|---|
| Visualizações (3.4K) | count `vaga_view` das vagas da empresa |
| Candidatos (842) | count distinct candidatos que interagiram/candidataram |
| Matches da IA (156) | count `match_generated` da empresa |
| Mensagens (48) | count threads/mensagens não lidas |
| Contratações por Mês | count `hire` agrupado por mês |
| Gestão de Vagas → Views/Interessados/Matches por vaga | filtra por `vaga_id` |
| Feed de Atividades | últimos N eventos relevantes, formatados |

### 6.3 Anti-fraude / higiene

Deduplicar `profile_view` e `vaga_view` por (ator, alvo, dia) para não inflar. Ignorar auto-visualização (candidato vendo o próprio perfil, empresa vendo a própria vaga). Bots/crawler não contam.

---

## 7. Motor 4 — Reputação e confiança

Camada que o Indeed tem solta e que o Vezulo pode integrar ao fluxo — crucial porque a empresa paga.

- **Selo Verificada (empresa):** do checklist do §4.2. Filtro e ordenação na busca do candidato.
- **Taxa de resposta (empresa):** % de conversas iniciadas por candidato/empresa que a empresa respondeu em ≤ 48 h, janela de 30 dias. Exibir como badge ("Responde rápido") quando ≥ 80%. Vem de eventos `message`.
- **Avaliação da empresa (novo, estilo Indeed):** nota 1–5 dada por candidatos que tiveram processo. Opcional na v1, mas é o principal ativo de confiança do Indeed — recomendo roadmap.
- **Status de disponibilidade (candidato):** `Disponível` / `Aberto a propostas` / `Empregado`. Alimenta a dimensão "atividade" do match e um filtro para a empresa. Substitui com vantagem o "visto por último" do Indeed.
- **Selo "Mais Procurado / Top 5%":** ver Motor 6.

---

## 8. Motor 5 — Economia de Vezus

O coração da monetização. Um Vezu desbloqueia um perfil/currículo.

### 8.1 Pacotes (da tela Financeiro)

| Pacote | Vezus | Preço | Preço/Vezu |
|---|---:|---:|---:|
| Start | 10 | R$ 60 | R$ 6,00 |
| Growth (popular) | 30 | R$ 150 | R$ 5,00 |
| Pro | 100 | R$ 450 | R$ 4,50 |
| Enterprise | 500 | R$ 2.000 | R$ 4,00 |

Desconto por volume incentiva pacote maior. Valores em `option` editável (`vezulo_vezus_packages`).

### 8.2 O que consome Vezu

| Ação | Custo sugerido |
|---|---:|
| Desbloquear perfil / currículo em Buscar Talentos ou Favoritos | 1 Vezu |
| Ver dados de contato de candidato correspondente | 1 Vezu (ou incluído no desbloqueio) |
| Publicar vaga | 0 (grátis, para encher o funil) ou pacote à parte |
| Gerar ranking IA / Match Inteligente | 0 (grátis; monetiza no desbloqueio) |
| Enviar 1ª mensagem a candidato não desbloqueado | 1 Vezu (equivale a desbloquear) |

**Regra de ouro:** nunca cobrar Vezu por perfil de baixa qualidade. Antes de permitir desbloqueio, exigir que o candidato tenha perfil ≥ X% completo e verificado; caso contrário, não aparece como desbloqueável. Isso protege a percepção de valor do crédito.

### 8.3 Ledger de transações

Tabela `wp_vezulo_vezus_ledger`:

| Coluna | Descrição |
|---|---|
| `id` PK | |
| `company_id` | empresa |
| `delta` | +compra / −gasto / +estorno |
| `reason` | `purchase`, `unlock`, `message`, `refund` |
| `balance_after` | saldo resultante (auditoria) |
| `ref_id` | id do candidato/vaga/pacote relacionado |
| `created_at` | |

Saldo atual = último `balance_after` (ou `SUM(delta)`). Consumo mensal = `SUM(-delta)` dos gastos no mês. Estorno automático se o candidato desbloqueado estiver inativo/perfil removido em até 7 dias (protege a confiança).

### 8.4 ROI (diferencial vs. CPC do Indeed)

Métricas novas para o dashboard financeiro, que o Indeed não entrega de forma clara:

- **Vezus por contratação** = Vezus gastos ÷ contratações no período.
- **Taxa de conversão de desbloqueio** = contratações ÷ desbloqueios.
- **Custo médio por contratação** em reais = (Vezus por contratação × preço/Vezu).

Empresa que enxerga ROI compra mais.

---

## 9. Motor 6 — Gamificação e conquistas (candidato)

Da tela: "Conquistas" (Mais Procurado – Top 5% na sua área; Perfil Completo) e o card de "Próximos Passos".

| Conquista | Regra |
|---|---|
| Perfil Completo | completude = 100% |
| Mais Procurado / Top 5% | percentil de `profile_view` + `interest` nos últimos 30 dias dentro do mesmo cargo/área (coorte) ≥ 95 |
| Em Alta | visualizações da semana > semana anterior (a tela já diz "Seu perfil está em alta esta semana") |
| Match Master | ≥ 5 matches ≥ 90% |
| Verificado | e-mail + telefone confirmados |

Percentil por coorte evita comparar um farmacêutico com um dev. Recalcular diariamente via cron (`wp_schedule_event`).

---

## 10. KPIs de dashboard (consolidação)

### 10.1 Candidato (Dashboard)
Visualizações, Empresas interessadas, Matches realizados, Perfil Completo %, Evolução do Perfil (série 7d), Conquistas, Próximos Passos. Todos derivam dos Motores 1, 2, 3 e 6.

### 10.2 Empresa (Dashboard)
Saldo de Vezus, Visualizações, Candidatos, Matches da IA, Mensagens, Contratações por Mês (série 6m), Feed de Atividades, Vezus consumidos. Derivam dos Motores 3 e 5.

### 10.3 Gestão de Vagas (empresa)
Por vaga: status (Ativa/Pausada), Views, Interessados, Matches. Agregados: total de visualizações, candidatos interessados, matches realizados. Funil completo: **Views → Interessados → Candidaturas → Matches → Contratação** (espelha o funil do Indeed, mas com match no meio).

---

## 11. Implementação no WordPress (headless)

Segue o padrão modular já usado (`vezulo/NN-*.php` carregados pelo `vezulo-bootstrap.php`). Novos módulos sugeridos:

| Módulo | Responsabilidade |
|---|---|
| `09-schema.php` | cria as 3 tabelas custom (`events`, `match`, `vezus_ledger`) via `dbDelta` no `register_activation`/versão de schema em `option` |
| `10-events.php` | `POST /events` (registro) + helpers de agregação (contadores) |
| `11-metrics-candidate.php` | `GET /candidate/metrics` (completude, contadores, evolução, conquistas, próximos passos) |
| `12-metrics-company.php` | `GET /company/metrics` + `GET /vagas/{id}/metrics` |
| `13-match.php` | motor de match + cache; `GET /match/candidate`, `POST /match/company` |
| `14-vezus.php` | `GET /vezus/balance`, `GET /vezus/ledger`, `POST /vezus/purchase`, `POST /company/unlock/{candidate_id}` |
| `15-reputation.php` | taxa de resposta, verificação, status de disponibilidade |

**Cuidados herdados do backend atual:** gravar JSON com `wp_slash()` (acentos), checagens de ownership/role retornando 403, JWT no header `Authorization` (já validado na Hostinger sem `.htaccess`), CORS allow-list (adicionar domínio de produção em `04-cors.php`).

**Cálculos pesados (percentil, match em lote):** via `wp_schedule_event` (cron) gravando em cache/tabela, nunca no request do usuário. Contadores simples: query direta com índices. Considerar `transient` para os cards do dashboard (TTL 5–15 min).

**Endpoints de escrita sensíveis** (`unlock`, `purchase`): idempotência por chave de requisição e transação no ledger antes de liberar o dado, para nunca debitar sem entregar (ou entregar sem debitar).

---

## 12. Resumo executivo das melhorias sobre o Indeed

1. **Match explicável** nos dois lados — acaba com o "não selecionado" opaco.
2. **Status de disponibilidade** do candidato como peso de match e filtro — melhor que "visto por último".
3. **Reputação integrada ao fluxo** (verificação + taxa de resposta + avaliação da empresa) para proteger quem investe tempo/Vezus.
4. **Sinais de qualidade antes de cobrar Vezu** — o crédito só se gasta em perfil completo/verificado/ativo.
5. **ROI transparente** (Vezus por contratação) no lugar do CPC difícil de prever.
6. **Gamificação por coorte** (Top 5% na sua área) que puxa o candidato a completar o perfil — o que, por sua vez, aumenta a qualidade que a empresa paga para ver.

O fio condutor: **qualidade do perfil do candidato → confiança no match → valor percebido do Vezu → receita.** Cada motor reforça o próximo.
