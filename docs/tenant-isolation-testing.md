# Testando isolamento entre tenants

Isolamento multi-tenant é uma fronteira de segurança. O teste útil não pergunta apenas se o tenant correto consegue acessar seus dados; ele tenta provar que um tenant **não consegue** atravessar essa fronteira por leitura, escrita, binding de rota, jobs, cache, storage ou caminhos administrativos.

Este playbook é independente de pacote de tenancy. Adapte os exemplos ao mecanismo usado pelo projeto para resolver e propagar o tenant, mas preserve os invariantes e os casos negativos.

## Invariante principal

Para qualquer operação protegida, o tenant efetivo deve ser resolvido de uma fonte confiável e todo acesso deve permanecer limitado a esse contexto.

Uma suíte mínima deve provar simultaneamente que:

1. o tenant A consegue executar a operação permitida sobre seus próprios recursos;
2. o tenant A não consegue ler, alterar ou excluir recursos do tenant B;
3. a ausência de contexto de tenant falha de forma fechada;
4. o limite continua válido fora da request HTTP, especialmente em filas, cache e storage;
5. caminhos administrativos que atravessam tenants são explícitos, autorizados e auditáveis.

Não use `withoutGlobalScopes()`, `withoutMiddleware()` ou desabilite policies apenas para facilitar o teste. Isso remove justamente controles que a suíte precisa provar.

## Onde cada teste pertence

| Nível | O que provar | Exemplos |
|---|---|---|
| Unit | Componentes que constroem ou validam contexto e namespace | resolver de tenant, cache key builder, storage path builder, payload de job |
| Feature | Comportamento observável pela aplicação com autenticação e autorização reais | leitura, update, delete, route/model binding, busca, export, impersonation |
| Integration | Fronteiras que dependem de infraestrutura ou processos separados | worker de fila, troca de conexão, cache compartilhado, filesystem/object storage |

Prefira feature tests para a maior parte da prova de isolamento. Unit tests ajudam em componentes determinísticos, mas não substituem a verificação do fluxo real com autenticação, binding, policies e queries.

## Fixture mínima: dois tenants

Todo teste de recurso multi-tenant deve conseguir criar pelo menos dois tenants distintos e recursos equivalentes em cada um.

Exemplo ilustrativo:

```php
$tenantA = Tenant::factory()->create();
$tenantB = Tenant::factory()->create();

$userA = User::factory()->for($tenantA)->create();

$projectA = Project::factory()->for($tenantA)->create();
$projectB = Project::factory()->for($tenantB)->create();
```

O detalhe de `for($tenant)` varia conforme o modelo da aplicação. O importante é que o fixture torne inequívoco qual recurso pertence a cada tenant.

## 1. Leitura cross-tenant

Teste primeiro o caminho permitido e depois o negado. Isso evita um falso positivo em que tudo retorna erro porque a rota, autenticação ou fixture está quebrado.

```php
public function test_user_can_read_a_project_from_their_tenant(): void
{
    [$tenantA, $userA, $projectA] = $this->tenantFixture();

    $this->actingAs($userA)
        ->withTenant($tenantA)
        ->get("/projects/{$projectA->id}")
        ->assertOk();
}

public function test_user_cannot_read_a_project_from_another_tenant(): void
{
    [$tenantA, $userA] = $this->tenantFixture();
    $projectB = $this->projectForAnotherTenant();

    $this->actingAs($userA)
        ->withTenant($tenantA)
        ->get("/projects/{$projectB->id}")
        ->assertStatus(404); // ou 403, conforme o contrato do produto
}
```

Escolha `404` ou `403` de forma intencional. Muitos produtos retornam `404` para não revelar a existência de um identificador pertencente a outro tenant. O importante é que a resposta não entregue o recurso nem dados derivados dele.

Além da página de detalhe, cubra consultas indiretas:

- listagens e filtros;
- buscas por ID externo, slug ou UUID;
- autocomplete;
- endpoints de exportação;
- contagens e agregações;
- relacionamentos aninhados.

## 2. Update e delete cross-tenant

Uma consulta de leitura bem isolada não garante que mutations estejam protegidas.

```php
public function test_user_cannot_update_another_tenants_project(): void
{
    [$tenantA, $userA] = $this->tenantFixture();
    $projectB = $this->projectForAnotherTenant();

    $this->actingAs($userA)
        ->withTenant($tenantA)
        ->patch("/projects/{$projectB->id}", [
            'name' => 'changed-by-tenant-a',
        ])
        ->assertStatus(404);

    $this->assertDatabaseMissing('projects', [
        'id' => $projectB->id,
        'name' => 'changed-by-tenant-a',
    ]);
}
```

Para delete, confirme também o estado persistido depois da resposta negada. Um status HTTP correto não compensa uma mutation que ocorreu antes da autorização.

## 3. Route/model binding não pode furar o escopo

Laravel pode resolver um model antes de parte da lógica do controller. Teste o endpoint real para garantir que o binding respeita o tenant ou que a policy bloqueia o objeto resolvido.

Casos mínimos:

- ID válido do tenant A → permitido;
- ID válido do tenant B → negado;
- slug/UUID alternativo do tenant B → negado;
- nested binding, como `/projects/{project}/tasks/{task}` → ambos os níveis precisam pertencer ao tenant correto.

Não teste apenas um repository isolado se a aplicação usa implicit model binding em produção.

## 4. Ausência de contexto deve falhar fechada

Um dos testes mais valiosos é remover o contexto esperado e provar que a aplicação **não** cai para uma consulta global.

```php
public function test_protected_query_fails_when_tenant_context_is_missing(): void
{
    $project = Project::factory()->create();

    TenantContext::clear();

    $this->expectException(MissingTenantContext::class);

    app(ProjectFinder::class)->findOrFail($project->id);
}
```

A forma da falha depende da arquitetura: exception, `403`, `404` ou interrupção explícita. Evite defaults que equivalem a “sem tenant = todos os tenants”.

## 5. Jobs precisam restaurar e validar o tenant

Workers vivem além da request que criou o job. O contexto deve viajar no payload por um identificador estável e ser restaurado explicitamente antes do trabalho de negócio.

Teste pelo menos:

- job do tenant A processa somente dados do tenant A;
- tenant inexistente/suspenso não faz o job cair para contexto global;
- payload sem tenant é rejeitado quando a operação exige tenant;
- retries preservam o mesmo contexto;
- um job do tenant A seguido de um job do tenant B no mesmo worker não herda estado residual.

Exemplo conceitual:

```php
public function handle(TenantRepository $tenants, TenantContext $context): void
{
    $tenant = $tenants->findActiveOrFail($this->tenantId);

    $context->run($tenant, function (): void {
        // trabalho limitado ao tenant restaurado
    });
}
```

Um integration test é apropriado quando a aplicação altera conexão de banco, middleware de queue ou estado do worker. Nesses casos, execute o job pelo mesmo pipeline usado em produção sempre que possível, em vez de chamar apenas `handle()` diretamente.

## 6. Cache deve carregar o namespace do tenant

Um cache key sem tenant pode vazar dados mesmo quando o banco está perfeitamente filtrado.

Teste que chaves semanticamente iguais não colidem:

```php
$keyA = $keys->forTenant($tenantA, 'dashboard:summary');
$keyB = $keys->forTenant($tenantB, 'dashboard:summary');

$this->assertNotSame($keyA, $keyB);
```

Depois faça um feature/integration test que grave valores diferentes para A e B e confirme que cada contexto recupera apenas seu próprio valor.

Inclua também locks distribuídos, rate limits e caches de autorização se eles forem tenant-aware.

## 7. Storage, exports e arquivos temporários

Caminhos de arquivo devem ter fronteira equivalente à do banco.

Casos úteis:

- upload de A não aparece na listagem de B;
- download usando o identificador/path de B é negado para A;
- exports de A e B não reutilizam o mesmo caminho temporário;
- signed URLs não podem ser obtidas por um tenant para objeto de outro;
- cleanup de A não remove arquivos de B.

Quando houver object storage real ou adapter específico, mantenha unit tests no path builder e pelo menos um integration test no adapter usado pelo ambiente.

## 8. Impersonation e suporte administrativo

Impersonation é uma exceção privilegiada, não um atalho para ignorar tenant isolation.

Teste que:

- somente um papel autorizado inicia impersonation;
- o tenant alvo é explícito;
- o contexto normal do operador não é silenciosamente reutilizado;
- início e fim da sessão deixam evidência auditável sem registrar secrets;
- ao encerrar, o contexto anterior é limpo/restaurado corretamente;
- endpoints não autorizados continuam negados mesmo durante suporte, salvo exceções documentadas.

Se existir uma operação realmente cross-tenant, dê a ela uma policy/permission própria e teste esse privilégio diretamente. Não transforme `is_admin` em bypass universal implícito.

## Matriz mínima de regressão

Use esta tabela como gate para recursos críticos:

| Caso | Permitido | Negado | Nível recomendado |
|---|---:|---:|---|
| Ler recurso próprio | ✓ |  | Feature |
| Ler recurso de outro tenant |  | ✓ | Feature |
| Atualizar recurso de outro tenant |  | ✓ | Feature |
| Excluir recurso de outro tenant |  | ✓ | Feature |
| Binding por ID/slug/UUID de outro tenant |  | ✓ | Feature |
| Operar sem tenant quando tenant é obrigatório |  | ✓ | Unit + Feature |
| Job restaura tenant correto | ✓ |  | Integration |
| Job sem tenant válido |  | ✓ | Integration |
| Cache com mesma chave lógica entre A/B | ✓, isolado | colisão | Unit + Integration |
| Storage/export A versus B | ✓, isolado | vazamento | Feature + Integration |
| Impersonation autorizada | ✓ |  | Feature |
| Impersonation sem privilégio |  | ✓ | Feature |

## Casos que costumam escapar

Inclua regressões adicionais quando o produto usar:

- soft deletes;
- observers e model events;
- bulk actions;
- imports;
- exports assíncronos;
- scheduled commands;
- broadcast/websocket channels;
- notificações e e-mails que consultam dados depois do request;
- search indexes externos;
- analytics warehouses;
- feature flags por tenant.

A regra é simples: sempre que dados cruzarem uma fronteira de processo, storage ou serviço, verifique como o tenant é identificado e como a ausência/mismatch é tratada.

## Gate mínimo de CI

Para uma mudança que toca recurso multi-tenant, o pipeline deveria impedir merge se falhar qualquer teste de:

- happy path do tenant correto;
- leitura cross-tenant;
- mutation cross-tenant;
- contexto ausente;
- binding/authorization do endpoint alterado;
- fila/cache/storage quando a mudança toca essas fronteiras.

Não persiga cobertura percentual isoladamente. Um único teste negativo que tenta atravessar a fronteira pode valer mais que dezenas de assertions que exercitam apenas o happy path.

## Revisão antes do merge

Pergunte no review:

1. De onde vem o tenant efetivo e essa fonte é confiável?
2. Qual teste prova acesso permitido?
3. Qual teste prova acesso negado ao mesmo tipo de recurso de outro tenant?
4. O que acontece quando o contexto não existe?
5. Algum job, cache, arquivo, export, lock ou integração precisa do mesmo namespace?
6. Existe caminho administrativo? Ele é explícito, autorizado e auditável?
7. O teste usa o fluxo real ou remove middleware/scopes/policies que existem em produção?

O objetivo não é provar que um pacote de tenancy funciona. É provar que **a sua aplicação** mantém a fronteira em todos os caminhos relevantes.