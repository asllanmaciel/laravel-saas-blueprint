# Matriz de decisão para isolamento de tenants

Não existe uma estratégia universal de multi-tenancy. A escolha deve equilibrar risco de vazamento entre tenants, custo operacional, necessidade de restore granular, volume desigual de carga e maturidade da equipe.

Use esta matriz para registrar a decisão antes de escolher um pacote de tenancy ou modelar a infraestrutura.

## Comparação rápida

| Estratégia | Isolamento | Custo operacional | Restore de um tenant | Observabilidade | Risco de noisy neighbor | Melhor encaixe |
|---|---|---|---|---|---|---|
| Banco compartilhado com `tenant_id` | Lógico, aplicado pela aplicação e constraints | Baixo | Mais difícil: exige restore seletivo ou ferramentas próprias | Simples, mas precisa segmentar métricas por tenant | Alto a médio | MVPs, muitos tenants pequenos e equipe enxuta |
| Schema/namespace por tenant, quando suportado pelo banco | Lógico mais forte | Médio | Mais simples que no modelo totalmente compartilhado | Boa, desde que conexões e queries carreguem o tenant | Médio | Produtos com necessidade maior de separação sem operar muitos bancos |
| Banco por tenant | Forte limite operacional e de dados | Alto | Naturalmente granular | Mais complexa: muitas conexões, bancos e migrações | Baixo entre tenants | Clientes grandes, requisitos contratuais fortes ou workloads muito diferentes |

A tabela resume tendências, não garantias. Uma implementação ruim de qualquer estratégia ainda pode causar vazamento de dados, indisponibilidade ou restauração incompleta.

## 1. Banco compartilhado com `tenant_id`

Todas as linhas vivem no mesmo banco e as tabelas multi-tenant carregam um identificador do tenant.

### Vantagens

- menor custo de infraestrutura e operação;
- migrations simples e centralizadas;
- consultas agregadas e relatórios globais mais fáceis;
- funciona bem quando existem muitos tenants pequenos com comportamento semelhante.

### Risco principal

Uma query sem filtro de tenant pode atravessar o limite lógico e expor ou alterar dados de outro cliente.

### Mitigações mínimas

- `tenant_id` obrigatório nas tabelas multi-tenant;
- foreign keys e índices compostos que incluam o tenant quando fizer sentido;
- resolução central do tenant por request;
- policies de autorização além do filtro da query;
- testes sempre criando pelo menos dois tenants;
- namespace de tenant também em cache, storage, filas, exports e locks;
- revisão específica para queries administrativas que intencionalmente atravessam tenants.

### Sinais de que continua sendo uma boa escolha

- a equipe consegue testar o isolamento de forma consistente;
- restore seletivo é exceção, não rotina;
- a maioria dos tenants tem volume parecido;
- não existem exigências contratuais de infraestrutura dedicada;
- o banco ainda tem folga de capacidade e índices previsíveis.

## 2. Schema ou namespace por tenant

Cada tenant recebe uma área lógica separada dentro da mesma tecnologia de banco, quando essa capacidade existir e for adequada ao ambiente escolhido.

### Vantagens

- reduz a chance de uma consulta comum acessar dados de outro tenant;
- facilita algumas operações de exportação e manutenção por tenant;
- preserva parte da eficiência de operar uma infraestrutura compartilhada.

### Risco principal

O número de schemas ou namespaces pode crescer mais rápido que a capacidade operacional da equipe. Migrations, inspeção de performance e troubleshooting passam a precisar de automação por tenant.

### Mitigações mínimas

- registrar a versão de schema aplicada por tenant;
- automatizar migrations com retomada segura após falha;
- limitar concorrência de operações administrativas em massa;
- manter métricas de erro, duração e atraso de migration;
- testar criação, atualização e remoção de tenants em ambiente descartável;
- definir desde cedo como backups e restores localizam um tenant específico.

### Sinais para adotar ou abandonar

Considere essa estratégia quando o compartilhamento total estiver limitando restore, manutenção ou separação lógica, mas um banco por tenant ainda trouxer custo operacional desproporcional.

Considere migrar além dela quando tenants precisarem de regiões diferentes, janelas próprias de manutenção, restauração isolada frequente ou capacidade de banco dedicada.

## 3. Banco por tenant

Cada tenant recebe um banco independente. A aplicação precisa resolver o tenant e selecionar a conexão correta antes de executar operações de negócio.

### Vantagens

- blast radius menor para falhas e operações de dados;
- backup, restore e exportação naturalmente mais granulares;
- permite dimensionar tenants grandes de forma independente;
- facilita requisitos contratuais de separação de dados em alguns cenários.

### Risco principal

O ganho de isolamento troca complexidade de aplicação por complexidade operacional: conexões, migrations, observabilidade, backups e rotação de credenciais passam a existir em escala multiplicada pelo número de tenants.

### Mitigações mínimas

- catálogo central de tenants e conexões sem secrets em texto aberto;
- pool e limites de conexão bem definidos;
- migrations automatizadas, versionadas e idempotentes;
- inventário verificável de bancos ativos, suspensos e em processo de exclusão;
- dashboards que permitam agregar métricas sem esconder outliers;
- restore drill periódico de um único tenant;
- processo explícito para mover um tenant entre clusters ou regiões.

### Sinais de que o custo se justifica

- poucos tenants muito grandes ou com workloads muito diferentes;
- restauração individual é requisito frequente;
- contratos exigem limites operacionais mais fortes;
- determinados tenants precisam de capacidade, região ou manutenção independentes;
- a equipe já possui automação madura para provisionamento, migration, backup e observabilidade.

## Critérios que devem pesar na decisão

### Isolamento e impacto de falha

Pergunte qual é o pior cenário real se uma query, migration ou operação administrativa estiver errada. Quanto maior o impacto potencial e menor a tolerância a vazamento ou indisponibilidade, mais forte deve ser o limite técnico e operacional.

### Backup e restore

Backup global é diferente de restore granular. Antes de escolher a estratégia, descreva como recuperar apenas um tenant, em quanto tempo e como validar que nenhum outro tenant foi alterado durante o processo.

### Noisy neighbor

Tenants com grande volume de jobs, consultas pesadas ou picos de importação podem disputar CPU, I/O, conexões e cache com os demais. A estratégia de dados não resolve tudo sozinha; quotas, filas, rate limits e observabilidade também precisam carregar o contexto do tenant.

### Observabilidade

Logs, métricas e traces devem responder perguntas como:

- qual tenant concentrou erros;
- qual tenant gerou crescimento de fila;
- qual operação degradou o banco;
- a falha afetou todos ou apenas um subconjunto de tenants.

Evite colocar dados pessoais ou secrets em labels de alta cardinalidade.

### Requisitos regulatórios e contratuais

Alguns mercados ou contratos podem exigir retenção, residência, exportação, exclusão ou separação específica de dados. Trate isso como requisito de arquitetura a ser validado com as áreas responsáveis; este blueprint não substitui análise jurídica, regulatória ou de compliance.

## Gatilhos de migração

Uma arquitetura saudável define antecipadamente quais sinais justificam mudar de estratégia.

### Compartilhado → isolamento maior

Considere evoluir quando um ou mais sinais aparecerem:

- restore de um único tenant começa a ser recorrente;
- tenants grandes causam contenção perceptível nos demais;
- surgem contratos com requisitos de isolamento incompatíveis com o modelo atual;
- manutenção global passa a ter blast radius alto demais;
- a equipe precisa aplicar limites de capacidade diferentes por tenant.

### Schema → banco por tenant

Considere quando:

- o número de schemas torna migrations e troubleshooting difíceis de operar;
- tenants precisam de regiões ou janelas de manutenção distintas;
- restore realmente independente vira requisito central;
- capacidade e performance precisam ser dimensionadas por cliente.

## Perguntas para registrar no ADR

Antes de fechar a decisão, responda:

1. Quantos tenants esperamos em 12 e 24 meses?
2. Qual a diferença esperada entre o menor e o maior tenant?
3. Qual é o RPO/RTO necessário para um único tenant?
4. Precisamos restaurar um tenant sem tocar nos demais?
5. Existem requisitos de região, retenção ou separação contratual?
6. Como migrations serão aplicadas e retomadas após falha?
7. Como jobs, cache, storage e exports recebem o contexto do tenant?
8. Qual métrica indicará que a estratégia atual deixou de ser suficiente?
9. Qual é o plano de migração se esse gatilho for atingido?

## Regra prática

Comece pela opção mais simples que preserve os limites de segurança e operação do produto, mas documente os gatilhos que obrigariam uma migração futura. Simplicidade sem critério vira dívida; isolamento máximo sem necessidade vira custo operacional permanente.
