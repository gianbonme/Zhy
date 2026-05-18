# 🚀 Jira Setup — Zhy: Do Zero à Escala
> **Versão:** 2.0 — Escalável para 20+ funcionários e alto volume
> Guia definitivo, personalizado para a operação atual e o futuro da Zhy.

---

## 🧠 Filosofia de Uso

O Jira é o **sistema nervoso central da Zhy** — não apenas uma lista de tarefas.

Ele integra 3 camadas que coexistem:

| Camada | Ferramenta | O que vive lá |
|:-------|:-----------|:--------------|
| **Estratégia & Contexto** | Repositório Git + IA | Formulações, narrativa, análise, plano de lançamento |
| **Execução & Coordenação** | **Jira** | Quem faz o quê, quando, bloqueios, progresso |
| **Dados & Finanças** | Google Sheets + Drive | Precificação, margens, documentos contratuais |

**Regra de ouro:** Se não está no Jira, não existe para a equipe.
**Regra de escala:** Projete hoje como se a empresa já tivesse 20 pessoas. O custo de reformatar depois é alto.

---

## 🗺️ Visão da Estrutura de Projetos (Escalável)

```
JIRA (Zhy)
│
├── ZHY-OPS    → Operação & Execução (Scrum — sprints semanais)
├── ZHY-ROAD   → Portfólio de Produtos & Lançamentos (Kanban — fluxo contínuo)
├── ZHY-COME   → Comercial & Vendas B2B/D2C (Kanban — pipeline de vendas)  ← futuro próximo
└── ZHY-MKTG   → Marketing & Branding (Scrum — ciclos de campanha)          ← futuro próximo
```

> **Hoje você começa com OPS + ROAD.** COME e MKTG são criados quando os respectivos líderes forem contratados. A estrutura já está planejada para isso.

---

## 📐 Hierarquia de Issues (Padrão em Todos os Projetos)

```
Initiative (Objetivo estratégico semestral)
  └── Epic (Frente de trabalho — semanas a meses)
        └── Story (Entrega com valor tangível — dias a semanas)
              └── Task / Sub-task (Ação concreta — horas a 1-2 dias)
```

**Exemplos reais da Zhy:**

| Nível | Exemplo |
|:------|:--------|
| Initiative | "Lançar Fase 1 até Jun/2026" |
| Epic | "Fechar parceiro HPP no RJ" |
| Story | "Validar Green People como co-packer de sucos" |
| Task | "Ligar para Green People — confirmação da reunião" |

---

## 🗂️ Projetos em Detalhe

---

### 📌 Projeto 1: `ZHY-OPS` — Operação & Execução
**Tipo:** Scrum · **Sprint:** 1 semana · **Start day:** Segunda-feira

#### Issue Types:
| Tipo | Uso |
|:-----|:----|
| **Initiative** | Objetivos macro do semestre (ex: "Lançar Fase 1", "Escalar Vendas B2B") |
| **Epic** | Frentes abertas (HPP, Embalagem, Contratações, etc.) |
| **Story** | Entregas com valor claro para a operação |
| **Task** | Ações atômicas executáveis |
| **Bug** | Problemas que surgem na operação |
| **Blocker** | Gargalos formais — linkados às Stories que bloqueiam |

#### Workflow:
```
Backlog → Sprint Atual → Em Andamento → Aguardando/Bloqueado → Em Revisão → Concluído
```

> O estado **"Aguardando/Bloqueado"** deve ter cor **vermelha** no board. É o sinal de alarme da operação.

---

### 📌 Projeto 2: `ZHY-ROAD` — Portfólio de Produtos
**Tipo:** Kanban · Sem sprints · Fluxo contínuo

#### Workflow:
```
Fase Futura → Planejando → P&D / Testes → Aprovação Regulatória → Pré-Lançamento → Lançado → Escala
```

#### Epics (um por linha de produto):
| Epic | Fase de Lançamento |
|:-----|:-------------------|
| ☕ Linha Café (Cápsulas + Moído) | Fase 1 — Jun/2026 |
| 🧃 Sucos Cold Press | Fase 1 — Jun/2026 |
| 💉 Shots Funcionais | Fase 2 — Out/2026 |
| 🍵 Solúveis / Lattes | Fase 3 — Jan/2027 |
| 🥫 RTD / Enlatados | Fase 4 — Mar/2027 |
| 💊 Suplementos Puros | Fase 5 — Longo Prazo |

---

### 📌 Projeto 3: `ZHY-COME` — Comercial & Vendas *(criar quando contratar B2B)*
**Tipo:** Kanban · Pipeline de vendas

#### Workflow (CRM simplificado dentro do Jira):
```
Lead → Contato Inicial → Reunião/Visita → Proposta Enviada → Negociação → Fechado ✅ / Perdido ❌
```

#### Campos customizados para esse projeto:
| Campo | Tipo |
|:------|:-----|
| **Canal** | Dropdown: B2B · D2C · Marketplace |
| **Tipo de PDV** | Dropdown: Academia · Empório · Cafeteria · Hotel · Farmácia · E-commerce |
| **Cidade** | Texto |
| **Valor Estimado (R$)** | Número |
| **Produto de Interesse** | Multi-select |
| **Próximo Follow-up** | Data |

> Quando a equipe comercial crescer, migre para um CRM dedicado (HubSpot ou Pipedrive). Mas por 1-2 anos, o Jira é suficiente.

---

### 📌 Projeto 4: `ZHY-MKTG` — Marketing & Branding *(criar quando Caco/Carol formalizarem)*
**Tipo:** Scrum · Sprint de 2 semanas (campanhas)

#### Issue Types:
| Tipo | Uso |
|:-----|:----|
| Epic | Campanha (ex: "Lançamento Fase 1 — Julho Redes Sociais") |
| Story | Entregável (ex: "Pack de 10 posts Instagram para sucos") |
| Task | Ação (ex: "Foto produto — suco maçã + gengibre") |

---

## 🏷️ Campos Customizados Globais

Crie esses campos como **globais** (disponíveis em todos os projetos):

| Campo | Tipo | Valores |
|:------|:-----|:--------|
| **Departamento** | Dropdown | Estratégia · Produto · Operação · Comercial · Marketing · Regulação · Tech · Financeiro · RH |
| **Parceiro/Fornecedor** | Dropdown | Vida Líquida · MushSpresso · Green People · Agência Rebu · Caco/Carol · Priscila (MAPA) · Adriana (ANVISA) · Guilherme Tech · Interno |
| **Criticidade** | Dropdown | 🔴 Crítico · 🟡 Alto · 🟢 Normal · ⚪ Baixo |
| **Fase do Produto** | Dropdown | Fase 1 · Fase 2 · Fase 3 · Fase 4 · Fase 5 |
| **Aguardando de** | Texto | Ex: "Resposta Green People sobre HPP" |
| **Deadline Real** | Data | Data de entrega comprometida (diferente do due date padrão) |

---

## 👥 Estrutura de Usuários & Times (Escalável)

### Grupos (criar agora, popular conforme cresce):

| Grupo Jira | Membros Atuais | Membros Futuros |
|:-----------|:---------------|:----------------|
| **Liderança** | Gianluca | + Gabriel Victor (COO), + CFO futuro |
| **Operação** | Guilherme RJ | + Técnicos, + Logística, + Qualidade |
| **Comercial** | (vaga aberta) | + 3-5 vendedores B2B, + CS |
| **Marketing** | Caco, Carol | + Social Media, + Performance, + Rebu |
| **Tech** | Guilherme Tech | + Dev, + Data |
| **Regulação** | Priscila, Adriana | + Consultor ANVISA futuro |
| **Parceiros** | Alvaro, Adriel | + Green People, + GEPEA |

### Permissões por grupo:

| Grupo | ZHY-OPS | ZHY-ROAD | ZHY-COME | ZHY-MKTG |
|:------|:-------:|:--------:|:--------:|:--------:|
| Liderança | Admin | Admin | Admin | Admin |
| Operação | Edit | View | View | View |
| Comercial | View | View | Edit | View |
| Marketing | View | View | View | Edit |
| Tech | Edit | View | View | View |
| Regulação | View | Edit (só seu Epic) | — | — |
| Parceiros | View (só seus cards) | — | — | — |

---

## 📊 Boards por Departamento (Para 20 pessoas)

### Board 1: 🎯 Board Estratégico (CEO/COO)
**Quem usa:** Gianluca + Gabriel Victor
**O que mostra:** Initiatives + Epics críticos + Blockers de todos os projetos
```
JQL: project in (OPS, ROAD, COME, MKTG) AND priority in (Critical, High) AND status != Concluído
```

### Board 2: ⚙️ Board Operacional (Equipe OPS)
**Quem usa:** Guilherme RJ + equipe de produção
**O que mostra:** Sprint atual de OPS — só tarefas de operação/produção
```
JQL: project = OPS AND Departamento = "Operação" AND sprint in openSprints()
```

### Board 3: 💼 Board Comercial (Vendas)
**Quem usa:** Equipe B2B
**O que mostra:** Pipeline de vendas completo em kanban
```
project = COME ORDER BY "Próximo Follow-up" ASC
```

### Board 4: 🎨 Board Marketing (Criação)
**Quem usa:** Caco, Carol, Rebu
**O que mostra:** Entregas criativas em andamento
```
project = MKTG AND sprint in openSprints()
```

### Board 5: 🔴 Board de Gargalos (Reunião de Segunda)
**Quem usa:** Toda a liderança, toda segunda-feira
**O que mostra:** Tudo bloqueado ou em risco em toda a empresa
```
project in (OPS, ROAD, COME, MKTG) AND status = "Aguardando/Bloqueado" ORDER BY Criticidade DESC
```

---

## 🏁 OKRs no Jira (Para Medir Sucesso)

Configure **Objectives** como Issues do tipo Initiative em `ZHY-OPS`:

### Q2 2026 (Abr → Jun):
| Objective | Key Results (como Epics) | Meta |
|:----------|:-------------------------|:-----|
| 🚀 Lançar Fase 1 | HPP fechado · PET definido · Site live · 100 unidades vendidas | Jun/01 |
| 👥 Estruturar a Equipe | COO formalizado · B2B contratado | Jun/30 |
| 🏭 Infraestrutura Operacional | Bombeiros OK · Lab encapsulamento funcional | Jun/30 |

### Q3 2026 (Jul → Set):
| Objective | Key Results | Meta |
|:----------|:------------|:-----|
| 📈 Tração Comercial | 10 PDVs B2B ativos · R$X de receita | Set/30 |
| 🧪 Fase 2 Ready | Shots formulados + parceiro fechado | Set/30 |
| 🔬 Conselho Científico | 3 membros formalizados | Ago/31 |

### Q4 2026 (Out → Dez):
| Objective | Key Results | Meta |
|:----------|:------------|:-----|
| 💰 4.000 Unidades | 3k sucos + 1k cápsulas vendidas | Dez/31 |
| 🚀 Lançar Fase 2 | Shots e Café Moído no mercado | Out/15 |
| 🏪 20 PDVs B2B | Rede consolidada no RJ | Dez/31 |

---

## 📅 Ritual de Gestão (Cadência Escalável)

### 🗓️ Daily (quando a equipe crescer acima de 5):
- 15 min em stand-up (presencial ou async no Slack)
- Cada pessoa reporta: Ontem fiz X · Hoje farei Y · Bloqueio: Z
- Qualquer bloqueio vira um card `Blocker` no Jira imediatamente

### 🗓️ Sprint Planning (toda Segunda — 30 min):
1. Abrir o **Board de Gargalos** — resolver ou redelegar blockers
2. Revisar sprint anterior — o que ficou para trás e por quê
3. Puxar tarefas do Backlog para a sprint da semana
4. Definir quem é dono de cada card

### 🗓️ Sprint Review (toda Sexta — 20 min):
1. Fechar sprint — mover não-concluídos
2. Atualizar `ZHY-ROAD` com progresso real das linhas de produto
3. Capturar 1 lição aprendida da semana (campo `Learnings` na sprint)

### 🗓️ Monthly Business Review (toda última Sexta do mês — 1h):
1. Revisar OKRs do mês — atingidos, em risco, perdidos
2. Atualizar o `ZHY-ROAD` com status de cada fase
3. Revisar pipeline comercial (`ZHY-COME`)
4. Prioridade do próximo mês

---

## 🔗 Integrações Essenciais

### GitHub ↔ Jira (Imediato)
- Instale **GitHub for Jira** (gratuito no Atlassian Marketplace)
- Repositório: `gianbonme/Zhy`
- Como usar: mencione `OPS-42` em commits/PRs → o card atualiza automaticamente
- Resultado: histórico de código ligado ao contexto operacional

### Slack ↔ Jira (Quando tiver equipe)
- Canal `#zhy-ops` recebe notificações de cards bloqueados e concluídos
- Canal `#zhy-gargalos` recebe alertas automáticos de Blockers abertos há mais de 48h
- Membros podem criar issues do Jira diretamente do Slack com `/jira create`

### Google Drive ↔ Jira
- Nos cards de regulação: linkar dossiês da Priscila e Adriana
- Nos cards de embalagem: linkar briefings e artes
- Nos cards comerciais: linkar contratos e propostas

### Automações Nativas do Jira (Configurar desde o início):
| Gatilho | Ação |
|:--------|:-----|
| Card parado em "Em Andamento" há 3+ dias | Envia alerta para o assignee + muda label para "Em Risco" |
| Card movido para "Bloqueado" | Notifica o Gianluca automaticamente |
| Sprint encerrada com >30% de cards não concluídos | Cria um card de retrospectiva automático |
| Issue com prioridade "Crítico" criada | Notifica toda a Liderança imediatamente |
| Novo lead criado em ZHY-COME | Cria automaticamente uma task de "Primeiro Contato" assignada ao B2B |

---

## 📈 Métricas & Reports (Para Gestão de Alto Nível)

Configure esses relatórios no Jira e revise mensalmente:

| Relatório | O que mostra | Frequência |
|:----------|:-------------|:-----------|
| **Velocity** | Quantas tasks a equipe fecha por sprint | Semanal |
| **Cumulative Flow** | Onde as tasks estão acumulando (gargalos sistêmicos) | Semanal |
| **Sprint Burndown** | Se a sprint vai ser concluída no prazo | Diário |
| **Epic Progress** | % de conclusão de cada Epic | Quinzenal |
| **Blockers Report** | Quantos dias cada blocker ficou aberto | Mensal |
| **Commercial Pipeline** | Quantos leads em cada etapa, valor total | Semanal |

---

## 🗓️ Sprint 1 — Issues Para Criar Agora (07/Mai/2026)

### Epic: 🔴 HPP & Infraestrutura Sucos
- `[STORY]` Fechar parceiro HPP no RJ
  - `[TASK]` Follow-up Green People (HPP sucos + shots) `[AGUARDANDO]`
  - `[TASK]` Mapear alternativas de HPP no RJ (plano B)
  - `[TASK]` Fechar orçamento rotuladora para Alvaro
  - `[TASK]` Comprar 2 freezers para Vida Líquida

### Epic: 🔴 Embalagem & PET
- `[STORY]` Definir garrafa PET para sucos e shots
  - `[TASK]` Aguardar lista de fornecedores PET do Alvaro `[AGUARDANDO]`
  - `[TASK]` Solicitar amostras: stand-up pouch + cápsulas
- `[STORY]` Resolver embalagem cápsulas café
  - `[TASK]` Aprovar arte embalagem (abertura pela tampa com logo Zhy)
  - `[TASK]` Cotar caixas com gráfica alternativa ao Adriel
- `[STORY]` Resolver embalagem café moído
  - `[TASK]` Criar briefing de material e design para a Rebu

### Epic: 🔴 Conselho Científico
- `[STORY]` Constituir conselho médico/nutricional
  - `[TASK]` Criar templates de mensagem de prospecção
  - `[TASK]` Contatar lead indicado pela Adriana (ANVISA)
  - `[TASK]` Montar lista de 10 candidatos ao conselho

### Epic: 🟡 Contratações & Equipe
- `[STORY]` Formalizar contratação de Gabriel Victor (COO)
  - `[TASK]` Definir escopo e remuneração
  - `[TASK]` Assinar contrato
- `[STORY]` Contratar Vendedor B2B
  - `[TASK]` Publicar vaga no LinkedIn + Catho
  - `[TASK]` Definir processo seletivo
- `[TASK]` Resolver uniformes/jalecos com Alsco (Julio) `[URGENTE]`

### Epic: 🟡 Regulação
- `[TASK]` Check-in com Priscila — questões pendentes MAPA
- `[TASK]` Acompanhar autorização bombeiros `[AGUARDANDO]`
- `[TASK]` Follow-up Adriana — status ANVISA suplementos

### Epic: 🟢 Testes Encapsulamento (Lab RJ)
- `[TASK]` Monitorar testes com Encapsuladora 00 (Guilherme RJ)
- `[TASK]` Avaliar resultado extrato Cordyceps + Juba de Leão vs. moído
- `[TASK]` Ajustar dossiers conforme densidade dos testes

---

## 🧩 Nomenclatura & Padrões (Para a Equipe)

Ao criar qualquer issue, seguir este padrão no título:

```
[PRODUTO] — [AÇÃO] — [RESPONSÁVEL SE EXTERNO]
```

Exemplos:
- ✅ `[SUCOS] — Fechar orçamento garrafa PET 500ml — Alvaro`
- ✅ `[CAPSULAS] — Aprovar arte final embalagem — Caco/Carol`
- ✅ `[SHOTS] — Enviar infos para Green People — Conselho GP`
- ❌ `Falar com Alvaro` (genérico demais)

---

## 📌 O Que NÃO Migrar Para o Jira

| Item | Fica onde |
|:-----|:----------|
| Formulações e receitas | Git (`/01_produtos`) |
| Análise estratégica profunda | Git (`/05_estrategia`) + IA |
| Cálculos financeiros e margens | Google Sheets |
| Documentos regulatórios | Google Drive |
| Contratos e NDA | Google Drive |
| Contexto histórico e decisões | Git + IA (aqui) |
| Plano de lançamento narrativo | Git (`/00_dashboard/plano-lançamento.md`) |

O Jira é o **"quem, o quê e quando"** — o **"por quê"** e o **"como profundo"** vivem no repositório e nas sessões comigo.

---

## 🛣️ Roadmap do Próprio Jira (Evolução Conforme Cresce)

| Fase Zhy | Equipe | O que adicionar no Jira |
|:---------|:-------|:------------------------|
| **Agora (Fase 1)** | 3-6 pessoas | OPS + ROAD · Sprints semanais · Automações básicas |
| **Fase 2 (Q3/2026)** | 8-12 pessoas | Criar ZHY-COME · Integrar Slack · Adicionar Daily |
| **Fase 3 (Q4/2026)** | 12-18 pessoas | Criar ZHY-MKTG · Ativar OKR tracking · Monthly Business Review |
| **Fase 4 (2027+)** | 20+ pessoas | Avaliar Jira Advanced · Portfólio Plans · Capacity planning · Migrar ZHY-COME para HubSpot |

---

## ✅ Checklist de Setup (Execute Nesta Ordem)

- [ ] **1.** Criar conta Jira (free plan — até 10 usuários, gratuito)
- [ ] **2.** Criar projeto `ZHY-OPS` (tipo: Scrum, sprint 1 semana)
- [ ] **3.** Criar projeto `ZHY-ROAD` (tipo: Kanban)
- [ ] **4.** Configurar Issue Types em cada projeto
- [ ] **5.** Criar os 6 Custom Fields globais (Departamento, Parceiro, Criticidade, Fase, Aguardando de, Deadline Real)
- [ ] **6.** Configurar Workflow com estado "Aguardando/Bloqueado" em vermelho
- [ ] **7.** Criar os Epics listados acima em OPS e ROAD
- [ ] **8.** Popular a Sprint 1 com os issues da seção acima
- [ ] **9.** Configurar as 5 Automações essenciais
- [ ] **10.** Convidar: Gianluca (admin), Gabriel Victor (PM), Guilherme Tech (dev)
- [ ] **11.** Instalar integração **GitHub for Jira** → linkar `gianbonme/Zhy`
- [ ] **12.** Criar os 5 Boards por departamento
- [ ] **13.** Criar os 3 Filtros salvos (Gargalos, Fase 1, Por Responsável)
- [ ] **14.** Criar as Initiatives de OKR do Q2 2026
- [ ] **15.** Fazer o onboarding de Gabriel Victor na ferramenta (primeira sprint juntos)

---

> **Nota:** O Jira free suporta até 10 usuários. Quando passar disso, o plano Standard custa ~$8/usuário/mês. Com 20 pessoas = ~$160/mês — completamente justificável para o tamanho de operação que a Zhy vai ter.
