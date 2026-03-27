<![CDATA[<div align="center">

# 📊 Documentação Operacional BI — CasaToro

**Indicadores Operations Center**

| 🏢 Cliente | ⚙️ Responsável | 📅 Data |
|:---:|:---:|:---:|
| Casa Toro | Sonar | Março 2026 |

---

</div>

## 📑 Sumário

| # | Seção | Ir para |
|:-:|-------|---------|
| 1 | 🗄️ Fontes de Dados e Linhagem | [Ver](#1--fontes-de-dados-e-linhagem) |
| 2 | 🔄 Regras de Refresh | [Ver](#2--regras-de-refresh) |
| 3 | 🔐 Perfis de Acesso e RLS | [Ver](#3--perfis-de-acesso-e-rls) |
| 4 | 🛠️ Troubleshooting | [Ver](#4--troubleshooting--resolva-rápido) |

---

## 1 · 🗄️ Fontes de Dados e Linhagem

### Parâmetros de Conexão

| Parâmetro | Valor | O que faz |
|-----------|-------|-----------|
| `ServerName` | `jd-prod-mysqldb01...rds.amazonaws.com` | Servidor MySQL na AWS |
| `DatabaseName` | `JDOC_ANALYTICS` | Banco de analytics |
| `DatabaseType` | `MySQL` | Tipo do banco |
| `Environment` | `Dev` ⚠️ | Ambiente ativo |
| `FusoHorarioOffset` | `-3` | Horário de Brasília |

> [!WARNING]
> **O parâmetro `Environment` está em `Dev`.** Antes de publicar em produção, **trocar para `Production`** e validar o gateway.

---

### 🔀 Caminho dos Dados — Da Origem ao Dashboard

```
  ┌─────────────────────────────────────────────────────┐
  │  🗃️  ① ORIGENS                                      │
  │                                                     │
  │  MySQL AWS (JDOC_ANALYTICS)     SharePoint (Excel)  │
  │  ├── f_machine_alerts           └── Versões de      │
  │  ├── f_machine_breadcrumbs          Software.xlsx   │
  │  ├── f_machine_measurements                         │
  │  ├── f_engine_hours             Geradas no modelo:  │
  │  ├── d_organizations            ├── dCalendário     │
  │  ├── d_equipments               └── _att_at         │
  │  ├── d_devices                                      │
  │  └── d_machine_measurement                          │
  └──────────────────────┬──────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────┐
  │  ⚙️  ② MODELO POWER BI                              │
  │                                                     │
  │  Power Query carrega → Relacionamentos →            │
  │  68 medidas DAX calculam os indicadores             │
  └──────────────────────┬──────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────┐
  │  ☁️  ③ POWER BI SERVICE                             │
  │                                                     │
  │  Gateway conecta ao MySQL → Refresh diário →        │
  │  Controle de acesso (workspace / RLS)               │
  └──────────────────────┬──────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────┐
  │  👥  ④ USUÁRIOS FINAIS                              │
  │                                                     │
  │  Diretoria · C-level · Equipe operacional           │
  │  (navegador ou app mobile)                          │
  └─────────────────────────────────────────────────────┘
```

---

### 📦 Tabelas do Modelo

#### 📈 Tabelas de Fato (dados operacionais)

| Tabela | Origem | O que armazena |
|--------|--------|----------------|
| `f_machine_alerts` | MySQL | Alertas das máquinas |
| `f_machine_breadcrumbs` | MySQL | Sinal / conectividade |
| `f_machine_measurements` | MySQL | Telemetria de tecnologias |
| `f_engine_hours` | MySQL | Vida útil do motor |

#### 📋 Tabelas de Dimensão (cadastros)

| Tabela | Origem | O que armazena |
|--------|--------|----------------|
| `d_organizations` | MySQL | Fazendas / clientes |
| `d_equipments` | MySQL | Máquinas |
| `d_devices` | MySQL | Dispositivos / modems |
| `d_machine_measurement` | MySQL | Metadados de tecnologias |

> **💡 Dica:** No banco de origem, a tabela se chama `d_equipment` (sem "s"). No modelo, aparece como `d_equipments`.

#### 🔧 Tabelas Auxiliares

| Tabela | Fonte | O que faz |
|--------|-------|-----------|
| `dCalendário` | DAX (calculada) | Calendário 2021 até ano seguinte |
| `d_versao_software` | Excel no SharePoint | Versões de firmware |
| `d_imagens` | Binário embutido | Imagens dos equipamentos |
| `_att_at` | Power Query (runtime) | Carimbo de "Última Atualização" |

---

### ⚡ Pontos de Falha

| 🔴 Componente | Problema | Impacto | Quem resolve |
|:---:|------------|---------|:---:|
| MySQL AWS | Instância fora do ar / senha expirada | ❌ Dashboard sem dados | Infra / AWS |
| Gateway | Serviço offline / credencial vencida | ❌ Refresh não executa | Resp. gateway |
| SharePoint | Excel movido ou sem permissão | ⚠️ Firmware sem dados | Equipe CasaToro |
| Parâmetro `Env` | `Dev` em produção | ⚠️ Dados do ambiente errado | Analista BI Sonar |

---

## 2 · 🔄 Regras de Refresh

### Configuração Rápida

| 📅 Frequência | 🌙 Janela | 🔁 Tipo | ⏱️ Timeout |
|:---:|:---:|:---:|:---:|
| Diária | 00h–06h (Brasília) | Full Refresh | 2 horas |

> O modelo **não tem incremental refresh**. Todas as tabelas são recarregadas por completo.

---

### 🔄 Ciclo do Refresh — Passo a Passo

```
  ⏰ AGENDAMENTO
  │  Horário programado chega (ex: 02h00)
  │  Power BI Service inicia o processo
  │
  ▼
  🔑 CREDENCIAIS
  │  Verifica usuário e senha do MySQL
  │  (armazenados no gateway)
  │
  ▼
  🌐 GATEWAY  ◄── ponte obrigatória
  │  Conecta ao MySQL na AWS (rede privada)
  │  ⚠️ Sem gateway = sem refresh
  │
  ▼
  📥 CARREGAMENTO
  │  SELECT * em todas as tabelas
  │  Dados retornam via gateway
  │
  ▼
  ┌──────────────────────────────────┐
  │  ✅ Sucesso                      │
  │  Dashboard atualizado,           │
  │  _att_at registra timestamp      │
  ├──────────────────────────────────┤
  │  ❌ Falha                        │
  │  E-mail de erro ao responsável,  │
  │  próximo ciclo no dia seguinte   │
  │  (ou refresh manual)             │
  └──────────────────────────────────┘
```

---

### ✅ Checklist Pós-Refresh

Verifique **antes** de liberar o dashboard:

| # | O que verificar | Onde | Resultado esperado |
|:-:|-----------------|------|--------------------|
| 1️⃣ | Data de "Última Atualização" | Header de qualquer aba | Data de **hoje** |
| 2️⃣ | Card `QtdMaquinas` | Aba Parque de Máquinas | Maior que zero |
| 3️⃣ | Card "Conectados" | Aba Conectividade | Maior que zero |
| 4️⃣ | Alertas recentes | Aba Alertas (últimos 7 dias) | Deve retornar alertas |
| 5️⃣ | Cards de firmware | Aba Gestão de Software | "Atualizado" + "Desatualizado" = Total |
| 6️⃣ | Tabela `d_versao_software` | Aba Gestão de Software | Com registros (sem branco) |
| 7️⃣ | Medições de telemetria | Aba IP Tratores (período recente) | RPM/combustível ≠ zero |

---

### 🚨 Sinais de Falha

| Status | Erro | O que fazer |
|:------:|------|-------------|
| 🔴 | `Data source credentials` | Regravar senha no Power BI Service → Configurações |
| 🔴 | `Gateway not reachable` | Reiniciar serviço do gateway no servidor |
| 🟠 | `Unable to connect to the server` | Verificar RDS na AWS + Security Group |
| 🟡 | Erro em `d_versao_software` | Verificar Excel no SharePoint (movido? permissão?) |
| 🟡 | Dados ok mas desatualizados | Checar parâmetro `Environment` (pode estar em `Dev`) |
| ⚪ | "Última Atualização" antiga | Refresh não executou — verificar histórico no Service |

---

### 👥 Responsáveis pelo Refresh

| Quem | Responsabilidade |
|------|------------------|
| **Analista BI Sonar** | Configurar/monitorar refresh, ajustar credenciais |
| **Infra / AWS** | Disponibilidade do RDS, liberar IPs no Security Group |
| **Owner SharePoint** | Manter `Versões de Software.xlsx` atualizada |

---

## 3 · 🔐 Perfis de Acesso e RLS

> [!NOTE]
> **Nenhuma regra de RLS está ativa atualmente.** O controle é feito pelo Power BI Service (workspace/relatório). A implementação de RLS por organização é recomendada.

### Perfis Recomendados

| Perfil | Quem | Abas | Filtro RLS |
|--------|------|------|:----------:|
| 🟣 **Admin BI** | Equipe Sonar | Todas (incluindo diagnóstico) | Nenhum |
| 🔵 **Gestor** | Diretoria / C-level | Todas as operacionais | Nenhum |
| 🟢 **Operador** | Equipe de cada fazenda | Parque, Alertas, Conectividade, Tecnologias, IP | ✅ Por organização |
| ⚪ **Visualizador** | Consulta geral | Capa, Parque, Alertas, Conectividade | Opcional |

---

### 🔒 Como o RLS Funciona (quando implementado)

```
  👤 Usuário faz login
  │  Power BI identifica o e-mail
  │  (ex: joao@casatoro.com)
  │
  ▼
  🔍 Tabela de Mapeamento
  │  Associa e-mail → organization_id
  │  (qual fazenda o usuário pode ver)
  │
  ▼
  🔒 Filtro Aplicado
  │  d_equipments filtrado por organization_id
  │  → todas as tabelas de fato herdam o filtro
  │
  ▼
  📊 Resultado
     Operador vê só SUA fazenda
     Gestor vê TODAS as fazendas
```

**Expressão DAX sugerida para a role:**

```dax
[organization_id] = USERPRINCIPALNAME()
```

Ou via tabela de mapeamento `usuário ↔ organization_id` para maior flexibilidade.

---

### 📝 Como Solicitar Acesso

| Passo | Ação |
|:-----:|------|
| **1** | Enviar chamado/e-mail para **Sonar** com: nome, e-mail corporativo, perfil desejado e organização(ões) |
| **2** | Analista Sonar valida com gestor CasaToro |
| **3** | Acesso concedido no Power BI Service |
| **4** | Se RLS ativo: usuário adicionado à role |
| **5** | Usuário recebe e-mail com link de acesso |

---

### 🔐 Dados Sensíveis

| Nível | Dado | Cuidado |
|:-----:|------|---------|
| 🔴 **Alta** | Credenciais MySQL | Ficam no gateway, **nunca** no .pbix |
| 🟠 **Média** | Localização das máquinas | Restringir por organização via RLS |
| 🟠 **Média** | Lista de organizações | Não expor para operadores de outras fazendas |
| 🟢 **Baixa** | Nº de série dos equipamentos | Disponível a todos os perfis |

---

## 4 · 🛠️ Troubleshooting — Resolva Rápido

---

### 🕐 Dashboard Sem Atualizar

> **Sintoma:** "Última Atualização" mostra data anterior a hoje.

| Passo | O que fazer |
|:-----:|-------------|
| **1** | Power BI Service → Workspace → Dataset → **Histórico de atualização** |
| **2** | Verificar se aparece "Com falha" ou "Não executado" |
| **3** | Se falhou: expandir o erro → identificar a tabela |
| **4** | Gateway offline? → **Reiniciar** o serviço no servidor |
| **5** | Credencial expirada? → **Regravar** senha em Configurações → Credenciais |
| **6** | Disparar **refresh manual** e monitorar |
| **7** | RDS inacessível? → Escalar para **infra** |

---

### 📭 Visuais Sem Dados (Em Branco)

> **Sintoma:** Cards ou tabelas aparecem vazios.

| Passo | O que fazer |
|:-----:|-------------|
| **1** | **Limpar todos os filtros** da página → dados apareceram? |
| **2** | Verificar **filtro de data** — abas como Alertas e Tecnologias exigem período |
| **3** | Aba Conectividade: a **data selecionada** é válida? |
| **4** | O **refresh do dia** foi concluído com sucesso? |
| **5** | Só firmware vazio? → Verificar **Excel no SharePoint** |

---

### 📊 KPI com Valor Divergente

> **Sintoma:** Número no dashboard difere de outra fonte.

| Passo | O que fazer |
|:-----:|-------------|
| **1** | Identificar a **medida exata** (consultar Dicionário de Métricas) |
| **2** | Verificar **filtros ativos** — mesma data, org e recorte da fonte de comparação? |
| **3** | Confirmar que o **refresh** do dia foi concluído |
| **4** | Medidas de %: o **denominador** está filtrado diferente? |
| **5** | Conectividade: só conta `request_status = SUCCESS` |
| **6** | Persistiu? Escalar com **print** do visual + fonte de comparação |

---

### 🚫 Acesso Negado / Dados Errados

> **Sintoma:** Usuário não entra ou vê dados de outra organização.

| Passo | O que fazer |
|:-----:|-------------|
| **1** | Usuário foi **adicionado** ao workspace/relatório? |
| **2** | O **perfil** atribuído está correto? |
| **3** | RLS ativo? Conferir **e-mail** na role e mapeamento |
| **4** | Testar com **"Exibir como"** no Power BI Desktop |
| **5** | Vê dados de outra org? → Revisar **expressão DAX** da role |
| **6** | Escalar com: nome, e-mail, perfil e comportamento observado |

---

### ⚡ Falhas de Gateway / Dataset

| Erro | Causa | Solução |
|------|-------|---------|
| `Data source credentials` | Senha MySQL expirada | Regravar no Power BI Service |
| `Gateway not reachable` | Serviço offline | Reiniciar o gateway no servidor |
| `Connection refused / timeout` | RDS fora do ar ou IP bloqueado | Console AWS + Security Group |
| `Unable to open document` | Excel movido no SharePoint | Verificar arquivo e atualizar caminho |
| `Expression.Error: MySQL` | Driver incompatível | Atualizar MySQL ODBC no gateway |
| Refresh parcial | Tabela específica falhou | Identificar no histórico de erros |

---

### 📞 Quando e Para Quem Escalar

| Situação | Escalar para | Canal |
|----------|:------------:|-------|
| Credencial / gateway offline | Analista BI Sonar | Chamado / e-mail |
| RDS fora do ar | Infra CasaToro / AWS | Chamado **urgente** |
| Excel desatualizado | Owner da planilha | E-mail direto |
| KPI divergente | Analista BI Sonar | Chamado + print |
| Novo acesso / RLS | Sonar + Gestor CasaToro | E-mail formal |
| Bug no visual | Analista BI Sonar | Chamado + reprodução |

---

### ⏱️ Tempo de Resposta (SLA)

| Severidade | Critério | Prazo |
|:----------:|----------|:-----:|
| 🔴 **Crítica** | Dashboard completamente sem dados | **4 horas** |
| 🟠 **Alta** | Uma aba sem dados ou KPI crítico divergente | **1 dia útil** |
| 🟡 **Média** | Visual individual com problema | **2 dias úteis** |
| 🟢 **Baixa** | Ajuste cosmético ou nova permissão | **3 dias úteis** |

---

<div align="center">

*Sonar · Março 2026 · v2.0*

</div>
]]>