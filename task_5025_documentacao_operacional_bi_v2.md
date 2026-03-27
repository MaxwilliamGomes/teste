# Documentação Operacional do BI — CasaToro
**Projeto:** Indicadores Operations Center  
**Cliente:** Casa Toro  
**Responsável:** Sonar  
 



---

## Sumário

1. [Fontes de Dados e Linhagem Analítica](#1-fontes-de-dados-e-linhagem-analítica)
2. [Regras de Refresh e Validações Pós-Atualização](#2-regras-de-refresh-e-validações-pós-atualização)
3. [Perfis de Acesso e Regras de RLS](#3-perfis-de-acesso-e-regras-de-rls)
4. [Checklist de Troubleshooting e Escalonamento](#4-checklist-de-troubleshooting-e-escalonamento)

---

## 1. Fontes de Dados e Linhagem Analítica

### 1.1 Parâmetros de Conexão do Modelo

Os parâmetros abaixo estão definidos diretamente no modelo Power BI (grupo `Parameters` no Power Query) e controlam toda a conectividade com o banco de dados:

| Parâmetro | Valor atual | Descrição |
|---|---|---|
| `ServerName` | `jd-prod-mysqldb01.cqgeh1ips9hu.us-west-2.rds.amazonaws.com` | Servidor MySQL hospedado na AWS RDS (us-west-2) |
| `DatabaseName` | `JDOC_ANALYTICS` | Banco de dados de analytics do Operations Center |
| `DatabaseType` | `MySQL` | Tipo do banco (suporta `SQL Server` ou `MySQL`) |
| `SchemaName` | ` ` (vazio) | Schema — não utilizado em MySQL; reservado para uso com SQL Server |
| `Environment` | `Dev` | Ambiente ativo. Valores possíveis: `Dev` ou `Production` |
| `FusoHorarioOffset` | `-3` | UTC-3 (horário de Brasília) — controla o carimbo de tempo exibido em "Última Atualização" |

> **⚠️ Atenção:** O parâmetro `Environment` está atualmente definido como `Dev`. Para publicação em produção, este parâmetro deve ser alterado para `Production` e o gateway de dados deve estar configurado para o servidor correto.

### 1.2 Arquitetura de Conexão

O modelo utiliza duas funções Power Query centralizadas para toda a ingestão de dados:

```
fnGetConnection(sqlText)
  └─> MySQL.Database(ServerName, DatabaseName, [Query = sqlText])

fnGetTable(tableName, optional whereClause)
  └─> fnGetConnection("SELECT * FROM " & tableName [WHERE ...])
```

Todas as tabelas de dados operacionais são carregadas via `fnGetTable`, garantindo ponto único de configuração. Qualquer troca de servidor ou banco impacta apenas os parâmetros — não as queries individuais.

### 1.3 Tabelas do Modelo e Suas Fontes

#### Tabelas operacionais (MySQL — `JDOC_ANALYTICS`)

| Tabela no modelo | Tabela na origem | Tipo | Descrição |
|---|---|---|---|
| `d_organizations` | `d_organizations` | Dimensão | Cadastro de organizações (clientes/fazendas) |
| `d_equipments` | `d_equipment` | Dimensão | Cadastro de equipamentos (máquinas) |
| `d_devices` | `d_devices` | Dimensão | Cadastro de dispositivos (modems, hardware) |
| `d_machine_measurement` | `d_machine_measurement` | Dimensão | Dicionário de medições — metadados das tecnologias |
| `f_machine_alerts` | `f_machine_alerts` | Fato | Alertas operacionais gerados pelas máquinas |
| `f_machine_breadcrumbs` | `f_machine_breadcrumbs` | Fato | Registros de sinal/conectividade por máquina |
| `f_machine_measurements` | `f_machine_measurements` | Fato | Medições de telemetria de tecnologias embarcadas |
| `f_engine_hours` | `f_engine_hours` | Fato | Horas de vida útil do motor por equipamento |

> **Nota sobre `d_equipments`:** o nome da tabela na origem é `d_equipment` (sem o `s`), mas o modelo exibe como `d_equipments`. Atenção a esta diferença em queries manuais.

#### Tabelas com fonte externa (fora do MySQL)

| Tabela no modelo | Fonte | Tipo | Descrição |
|---|---|---|---|
| `d_versao_software` | SharePoint — `Versões de Software.xlsx` | Excel via Web | Planilha de referência de versões de firmware mantida manualmente. URL: `sonarddcombr.sharepoint.com/sites/SonarDD/...` |
| `d_imagens` | Embutida no modelo (Binary) | Estática | Imagens de equipamentos comprimidas em Base64 — sem fonte externa |

#### Tabelas calculadas e auxiliares

| Tabela no modelo | Tipo | Descrição |
|---|---|---|
| `dCalendário` | Calculada (DAX) | Calendário gerado de 2021 até `YEAR(TODAY())+1` — não tem fonte externa |
| `_att_at` | Power Query (M) | Calcula e formata o carimbo de "Última Atualização" no fuso UTC-3. Não tem fonte de dados — usa `DateTimeZone.UtcNow()` |
| `_Medidas` | Calculada | Tabela vazia que hospeda as medidas DAX do modelo |

### 1.4 Fluxo Macro de Dados (Linhagem)

O diagrama abaixo mostra o caminho completo dos dados, da origem até o dashboard consumido pelos usuários. Há três origens distintas, todas convergindo para o modelo semântico no Power BI Desktop antes de serem publicadas no Service.

```
╔══════════════════════════════════════════════════════════════╗
║  ① ORIGENS DOS DADOS                                         ║
╠═══════════════════════════════╦══════════════════════════════╣
║  MySQL · AWS RDS (us-west-2)  ║  SharePoint (Excel)          ║
║  Banco: JDOC_ANALYTICS        ║  Versões de Software.xlsx    ║
║                               ║  (mantida manualmente)       ║
║  Tabelas de fato:             ║                              ║
║  · f_machine_alerts           ║  Sem fonte externa:          ║
║  · f_machine_breadcrumbs      ║  · dCalendário  (DAX)        ║
║  · f_machine_measurements     ║  · _att_at      (M/runtime)  ║
║  · f_engine_hours             ║                              ║
║                               ║                              ║
║  Tabelas de dimensão:         ║                              ║
║  · d_organizations            ║                              ║
║  · d_equipments               ║                              ║
║  · d_devices                  ║                              ║
║  · d_machine_measurement      ║                              ║
╚═══════════════════════════════╩══════════════════════════════╝
                        │                     │
                        │  Power Query         │  Power Query
                        │  fnGetTable()        │  Excel.Workbook()
                        ▼                     ▼
╔══════════════════════════════════════════════════════════════╗
║  ② MODELO SEMÂNTICO — Power BI Desktop (.pbix)              ║
║                                                              ║
║  · Relacionamentos entre tabelas                             ║
║  · Medidas DAX (Alertas, Conectividade, Tecnologias...)   ║
║  · Parâmetros de conexão (ServerName, Environment, etc.)     ║
║  · Funções Power Query (fnGetTable, fnGetConnection)         ║
╚══════════════════════════════════════════════════════════════╝
                        │
                        │  Publicação do .pbix
                        ▼
╔══════════════════════════════════════════════════════════════╗
║  ③ PUBLICAÇÃO — Power BI Service                            ║
║                                                              ║
║  · On-premises Data Gateway  ◄──── ponte para o MySQL AWS   ║
║  · Refresh agendado diário                                   ║
║  · Controle de acesso (workspace / RLS)                      ║
╚══════════════════════════════════════════════════════════════╝
                        │
                        │  Acesso via navegador ou app
                        ▼
╔══════════════════════════════════════════════════════════════╗
║  ④ CONSUMO — Usuários finais                                 ║
║                                                              ║
║  Diretoria · C-level · Equipe operacional                    ║
╚══════════════════════════════════════════════════════════════╝
```

**Pontos de atenção no fluxo:**

- O Gateway é obrigatório na etapa ③ — sem ele o Service não consegue alcançar o MySQL da AWS (rede privada). Se o Gateway estiver offline, o refresh falha inteiramente.
- As tabelas `dCalendário` e `_att_at` são geradas em runtime no modelo, sem depender de nenhuma fonte externa. Não falham por problema de conectividade.
- O parâmetro `Environment` controla qual banco o modelo acessa. Atualmente está em `Dev`.
- A tabela `d_versao_software` tem dependência exclusiva do SharePoint. Uma falha nela não impede o carregamento das demais tabelas, mas remove o status de firmware do dashboard.

### 1.5 Dependências Externas e Pontos de Falha

| Dependência | Ponto de falha | Impacto | Responsável |
|---|---|---|---|
| AWS RDS MySQL | Instância fora do ar, senha expirada, IP bloqueado no Security Group | Todas as tabelas operacionais falham — dashboard sem dados | Equipe de infra / AWS |
| On-premises Data Gateway | Gateway offline, credencial expirada, serviço parado | Refresh agendado no Power BI Service não executa | Responsável do gateway |
| SharePoint (Excel) | Arquivo movido, renomeado ou permissão revogada | `d_versao_software` falha — status de firmware fica sem dados | Equipe CasaToro (owner da planilha) |
| Parâmetro `Environment = Dev` | Publicação sem alterar para `Production` | Modelo aponta para ambiente errado | Analista BI Sonar |
| Fuso UTC-3 | Ajuste de horário de verão não tratado | "Última Atualização" exibe hora incorreta | Analista BI Sonar |

---

## 2. Regras de Refresh e Validações Pós-Atualização

### 2.1 Agenda e Janela de Atualização

| Parâmetro | Valor |
|---|---|
| Frequência | Diária |
| Janela recomendada | Madrugada (00h–06h, horário de Brasília) — menor concorrência no banco |
| Tipo de refresh | Full refresh (todas as tabelas recarregadas integralmente) |
| Timeout recomendado | 2 horas |
| Ambiente de execução | Power BI Service com On-premises Data Gateway (necessário para acesso ao MySQL na AWS) |

> **Observação:** O modelo não possui incremental refresh configurado. Todas as tabelas, incluindo fatos volumosos (`f_machine_measurements`, `f_machine_breadcrumbs`, `f_machine_alerts`), são recarregadas por completo a cada atualização.

**Ciclo completo do refresh agendado:**

```
┌─────────────────────────────────────────────────────────────┐
│  POWER BI SERVICE                                           │
│                                                             │
│  1. Horário agendado chega (ex: 02h00, horário de Brasília) │
│     │                                                       │
│     ▼                                                       │
│  2. Service verifica as credenciais armazenadas             │
│     (usuário e senha do MySQL configurados no gateway)      │
│     │                                                       │
│     ▼                                                       │
│  3. Solicitação enviada ao On-premises Data Gateway ────────┼──┐
└─────────────────────────────────────────────────────────────┘  │
                                                                  │
          ╔ BARREIRA DE REDE ══════════════════════════════╗     │
          ║ O Service não acessa o MySQL diretamente.      ║     │
          ║ O Gateway vive na mesma rede privada do banco. ║     │
          ╚════════════════════════════════════════════════╝     │
                                                                  │
┌─────────────────────────────────────────────────────────────┐  │
│  ON-PREMISES DATA GATEWAY                                   │◄─┘
│                                                             │
│  4. Gateway conecta ao MySQL (AWS RDS · JDOC_ANALYTICS)     │
│     │                                                       │
│     ▼                                                       │
│  5. Executa SELECT * em todas as tabelas do modelo          │
│     (full refresh — sem incremental)                        │
│     │                                                       │
│     ▼                                                       │
│  6. Dados retornam ao Service via Gateway                   │
└─────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│  RESULTADO                                                  │
│                                                             │
│  ✔ Sucesso → modelo atualizado, _att_at registra timestamp  │
│              dashboard disponível com dados do dia          │
│                                                             │
│  ✘ Falha   → e-mail de erro ao responsável                  │
│              histórico de refresh registra a causa          │
│              próximo ciclo ocorre no dia seguinte           │
│              (ou acionar refresh manual após correção)      │
└─────────────────────────────────────────────────────────────┘
          │
          └──────────────────────────────────────────────────►
                           ↻  repete no próximo dia agendado
```

### 2.2 Owners do Refresh

| Papel | Responsabilidade |
|---|---|
| **Analista BI Sonar** | Configurar e monitorar o refresh agendado no Power BI Service; ajustar credenciais quando necessário |
| **Infra / AWS (CasaToro ou Sonar)** | Garantir disponibilidade do RDS MySQL e liberar IPs do gateway no Security Group |
| **Owner da planilha SharePoint** | Manter `Versões de Software.xlsx` atualizada com novas versões de firmware |

### 2.3 Sinais de Falha no Refresh

| Sinal | O que pode indicar |
|---|---|
| Erro "Data source credentials" | Credencial do MySQL expirada ou alterada; regravar no Power BI Service |
| Erro "Gateway not reachable" | Gateway offline ou sem conectividade com o RDS |
| Erro "Unable to connect to the server" | IP do gateway bloqueado no Security Group AWS ou RDS fora do ar |
| Erro em `d_versao_software` apenas | Arquivo Excel no SharePoint movido, renomeado ou com permissão revogada |
| Refresh concluído mas dados desatualizados | Parâmetro `Environment` incorreto |
| "Última Atualização" mostra data antiga | Refresh não executou no dia — verificar histórico no Power BI Service |

### 2.4 Validações Mínimas Pós-Refresh

Após cada atualização, verificar os seguintes pontos antes de liberar o dashboard para os usuários:

| # | Validação | Como verificar | Status esperado |
|---|---|---|---|
| 1 | Data de "Última Atualização" | Verificar o header de qualquer aba do dashboard | Deve exibir a data do dia atual |
| 2 | Cards de Parque de Máquinas | Abrir aba "Parque de Máquinas" sem filtros | `QtdMaquinas` deve ser maior que zero |
| 3 | Aba Conectividade — card "Conectados" | Abrir aba Conectividade sem filtros | `QtdMaquinasConectadas v3` deve ser maior que zero |
| 4 | Aba Alertas — período recente | Filtrar últimos 7 dias na aba Alertas | Deve retornar alertas (salvo situação operacional excepcional) |
| 5 | Status de firmware | Abrir aba Gestão de Software | Cards "Atualizado" e "Desatualizado" devem somar ao total |
| 6 | Tabela `d_versao_software` | Verificar se há registros na aba Gestão de Software | Tabela de referência com dados — sem erro em branco |
| 7 | Tabelas de medição | Abrir aba IP Tratores com período recente | Valores de RPM, combustível ou utilização devem ser diferentes de zero |

### 2.5 Ações Iniciais de Correção

| Problema identificado | Ação |
|---|---|
| Credencial expirada | Acessar Power BI Service → Configurações → Credenciais do gateway → Regravar a senha do MySQL |
| Gateway offline | Verificar o serviço "On-premises Data Gateway" no servidor onde está instalado; reiniciar se necessário |
| RDS inacessível | Verificar status da instância no console AWS e confirmar regras de Security Group |
| Excel do SharePoint com erro | Reabrir o arquivo no SharePoint, verificar permissões, recriar o link no Power Query se necessário |
| `Environment = Dev` em produção | Abrir o .pbix, alterar o parâmetro para `Production`, salvar e republicar no Service |
| Dados com delay inesperado | Verificar se o pipeline de origem (ETL do Operations Center) concluiu antes da janela de refresh do BI |

---

## 3. Perfis de Acesso e Regras de RLS

### 3.1 Estado Atual do RLS

> **⚠️ Nota:** Nenhuma regra de Row-Level Security (RLS) foi identificada no modelo atual. O campo `organization_id` presente nas tabelas principais (`d_equipments`, `f_machine_alerts`, `f_machine_breadcrumbs`) é estruturalmente adequado para implementação de RLS por organização, mas a lógica ainda não está configurada.

O controle de acesso hoje é feito exclusivamente pelo Power BI Service (nível de workspace/relatório), não por RLS no modelo.

### 3.2 Perfis de Acesso Recomendados

| Perfil | Áreas atendidas | Acesso às abas | Filtro RLS sugerido |
|---|---|---|---|
| **Administrador BI** | Equipe Sonar | Todas as abas, incluindo auxiliares e diagnóstico | Sem restrição |
| **Gestor CasaToro** | Diretoria / C-level | Capa, Parque, Alertas, Conectividade, Gestão de Software, Tecnologias, IP | Sem restrição (visão de todas as organizações) |
| **Operador por Organização** | Equipe operacional de cada fazenda/organização | Parque, Alertas, Conectividade, Tecnologias, IP | `d_equipments[organization_id] = USERNAME()` — restrito à própria organização |
| **Visualizador** | Usuários de consulta geral | Capa, Parque, Alertas, Conectividade | Sem restrição ou restrito à organização |

### 3.3 Restrições por Perfil (Quando RLS for Implementado)

| Tabela | Campo de filtro recomendado | Lógica sugerida |
|---|---|---|
| `d_equipments` | `organization_id` | Filtro pela organização do usuário |
| `f_machine_alerts` | Via relacionamento com `d_equipments` | Propagação automática do filtro |
| `f_machine_breadcrumbs` | Via `d_equipments` | Propagação automática do filtro |
| `f_machine_measurements` | Via `d_equipments` | Propagação automática do filtro |
| `f_engine_hours` | Via `d_equipments` | Propagação automática do filtro |
| `d_organizations` | `organization_id` | Limitar exibição às organizações autorizadas |

> **Implementação recomendada:** criar uma role DAX no Power BI com expressão do tipo:
> ```dax
> [organization_id] = USERPRINCIPALNAME()
> ```
> ou via tabela de mapeamento `usuário ↔ organization_id` para maior flexibilidade.

### 3.4 Processo para Solicitar Ajuste de Permissão

1. O solicitante envia e-mail ou abre chamado para o responsável BI Sonar informando:
   - Nome completo e e-mail corporativo do usuário
   - Perfil desejado (Gestor, Operador, Visualizador)
   - Organização(ões) que deve ter acesso (se Operador por Organização)
2. O analista Sonar avalia a solicitação com o gestor CasaToro responsável
3. O acesso é concedido no Power BI Service (workspace ou relatório)
4. Quando RLS estiver implementado: o analista adiciona o usuário à role correspondente no Power BI Service → Segurança do modelo semântico
5. O usuário recebe confirmação por e-mail com o link de acesso ao relatório

### 3.5 Cuidados com Dados Sensíveis

| Dado | Tabela | Sensibilidade | Cuidado recomendado |
|---|---|---|---|
| Localização geográfica das máquinas | `f_machine_breadcrumbs` | Média — pode expor operações de campo | Restringir por organização via RLS |
| Lista de organizações e clientes | `d_organizations` | Média — dados comerciais | Não expor organizações de terceiros para operadores |
| Número de série dos equipamentos | `d_equipments[serial_number]` | Baixa | Disponível a todos os perfis autorizados |
| Credenciais do banco (MySQL) | Parâmetros do modelo | **Alta** | Nunca expor o `.pbix` sem proteção de senha; credenciais armazenadas no gateway, não no arquivo |
| URL do SharePoint com nome de tenant | Partição `d_versao_software` | Baixa | Manter o arquivo .pbix como confidencial interno |

---

## 4. Checklist de Troubleshooting e Escalonamento

### 4.1 Dashboard Sem Atualizar

**Sintoma:** "Última Atualização" exibe data anterior ao dia de hoje.

| Passo | Ação |
|---|---|
| 1 | Acessar Power BI Service → Workspace → Dataset → Histórico de atualização |
| 2 | Verificar se o refresh do dia aparece como "Com falha" ou "Não executado" |
| 3 | Se "Com falha": expandir o erro e identificar a tabela que falhou |
| 4 | Se gateway offline: verificar serviço no servidor do gateway e reiniciar |
| 5 | Se credencial expirada: regravar a senha em Configurações → Credenciais |
| 6 | Disparar refresh manual após correção e monitorar conclusão |
| 7 | Se RDS inacessível: escalar para infra (ver seção 4.6) |

---

### 4.2 Visual Sem Dados (Cards ou Tabelas em Branco)

**Sintoma:** Um ou mais visuais aparecem vazios, com zero ou sem valor.

| Passo | Ação |
|---|---|
| 1 | Verificar se há filtros ativos na página — limpar todos e observar se os dados aparecem |
| 2 | Verificar o filtro de data: algumas abas exigem período definido (`Alertas`, `Tecnologias`, `IP`) |
| 3 | Na aba Conectividade: verificar se o filtro "Data" está definido para uma data válida |
| 4 | Se o problema persistir sem filtros: verificar se o refresh do dia foi concluído com sucesso |
| 5 | Verificar se a tabela de origem está populada: executar query de validação diretamente no banco (ver seção 4.7) |
| 6 | Se apenas `d_versao_software` estiver vazia: verificar o Excel no SharePoint |

---

### 4.3 Divergência de KPI

**Sintoma:** Valor exibido no dashboard difere do esperado ou de outra fonte de referência.

| Passo | Ação |
|---|---|
| 1 | Identificar a medida exata que apresenta divergência (consultar o Dicionário de Métricas) |
| 2 | Verificar quais filtros estão ativos na página no momento da divergência |
| 3 | Confirmar que a fonte de comparação usa o mesmo período e recorte (organização, família, modelo) |
| 4 | Verificar se o refresh do dia foi concluído — dados defasados causam divergência com fontes em tempo real |
| 5 | Para medidas de percentual (`QtdMaquinasConectadas`, `% Hoje v3`, etc.): confirmar que o denominador não está filtrado de forma diferente |
| 6 | Para medidas de conectividade: lembrar que faixas de sinal usam `request_status = SUCCESS` — máquinas "sem acesso" são excluídas |
| 7 | Se a divergência persistir após validação: escalar para o analista BI Sonar com print do visual e da fonte de comparação |

---

### 4.4 Acesso Negado / RLS Incorreto

**Sintoma:** Usuário não consegue acessar o relatório, ou vê dados incorretos (de outras organizações ou sem dados).

| Passo | Ação |
|---|---|
| 1 | Verificar se o usuário foi adicionado ao workspace ou ao relatório compartilhado no Power BI Service |
| 2 | Verificar se o perfil atribuído corresponde ao acesso esperado |
| 3 | Se RLS estiver ativo: verificar se o e-mail do usuário está correto na role e se o mapeamento de organização está atualizado |
| 4 | Testar a role usando a funcionalidade "Exibir como" no Power BI Desktop antes de publicar |
| 5 | Se o usuário vê dados de outra organização: revisar a expressão DAX da role de RLS |
| 6 | Se o usuário não vê dado nenhum com RLS ativo: verificar se o `organization_id` do usuário está mapeado corretamente |
| 7 | Escalar para analista BI Sonar com: nome do usuário, e-mail, perfil esperado e descrição do comportamento observado |

---

### 4.5 Falha em Dataset ou Gateway

**Sintoma:** Refresh falha com erros de conexão, timeout ou gateway.

| Código / Mensagem de erro | Causa provável | Ação |
|---|---|---|
| `Data source credentials` | Senha do MySQL alterada ou expirada | Regravar credencial no Power BI Service |
| `Gateway not reachable` | Serviço do gateway offline | Reiniciar o serviço "On-premises Data Gateway" no servidor |
| `Connection refused` / `timeout` | RDS fora do ar ou IP bloqueado | Verificar status no console AWS e regras do Security Group |
| `Unable to open document` (SharePoint) | Excel movido ou permissão revogada | Verificar o arquivo no SharePoint e atualizar o caminho no Power Query se necessário |
| `Expression.Error: MySQL` | Versão do driver MySQL no gateway incompatível | Atualizar o driver MySQL ODBC no servidor do gateway |
| Refresh parcial (algumas tabelas ok) | Tabela específica com problema — ver mensagem detalhada | Identificar a tabela no histórico de erros e corrigir pontualmente |

---

### 4.6 Quando Escalar e Para Quem

| Situação | Escalado para | Canal |
|---|---|---|
| Credencial expirada / gateway offline | Analista BI Sonar | Chamado interno / e-mail |
| RDS inacessível (instância AWS fora do ar) | Equipe de infra CasaToro ou AWS Support | Chamado urgente / telefone |
| Excel de versões de software desatualizado | Owner da planilha SharePoint (CasaToro) | E-mail direto |
| Divergência de KPI não explicada por filtro | Analista BI Sonar → revisão de lógica DAX | Chamado com evidência (print + filtros) |
| Solicitação de novo acesso ou ajuste de RLS | Analista BI Sonar + Gestor CasaToro | E-mail formal com aprovação do gestor |
| Bug no visual / comportamento inesperado | Analista BI Sonar | Chamado com descrição e reprodução do problema |

**Tempo de resposta sugerido:**

| Severidade | Critério | SLA |
|---|---|---|
| **Crítica** | Dashboard completamente sem dados em dia de uso | Até 4 horas |
| **Alta** | Uma aba específica sem dados ou KPI crítico divergente | Até 1 dia útil |
| **Média** | Visual individual com problema, filtro com comportamento inesperado | Até 2 dias úteis |
| **Baixa** | Ajuste cosmético, nova permissão de acesso | Até 3 dias úteis |

---



