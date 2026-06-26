# Repository Conventions

How we structure data access in our Laravel apps. Two kinds of repositories: **transactional** (CRUD on a single model) and **analytics** (read-only queries across joins).

---

## 1. Transactional repositories

One repo per Eloquent model. Returns Eloquent models.

**Naming:** `{Model}Repository` → `UserRepository`, `OrderRepository`

**Methods** describe persistence intent.

```php
class UserRepository
{
    public function find(int $id): ?User;
    public function findOrFail(int $id): User;
    public function all(): Collection;
    public function paginate(int $perPage = 15): LengthAwarePaginator;
    public function create(array $data): User;
    public function update(User $user, array $data): User;
    public function delete(User $user): bool;
    ...
}
```

**Query helpers** are the model-specific lookups beyond basic CRUD. Name them after the question they answer, not the SQL behind them. 
Use prefixes that signal the return type: `find...` (model or null), `findAll...`/plural (collection), `exists...` (bool), `count...` (int).

```php
class UserRepository
{
    ...
    public function findActiveByTeam(int $teamId): ?User;
    public function existsByEmail(string $email): bool;
}
```

### Need data from multiple models? Decide by ownership:

- **One clear primary entity, related data enriches it** → keep it in that entity's repo and let Eloquent load the relations. The repo still returns the primary model (with relations attached), not a flat join.

  ```php
  // OrderRepository — Order is the owner; customer/items are its relations
  public function findWithDetails(int $id): ?Order
  {
      return Order::with(['customer', 'items.product'])->find($id);
  }
  ```

- **No single owner — you're combining entities to answer a question** → that's not transactional. It belongs in an analytics repo (section 2), returns a DTO/array, and uses the query builder.

Rule of thumb: if the result still maps cleanly to *one* model, use relations in that model's repo. If it maps to no model, it's analytics.

---

## 2. Analytics repositories

One repo per **reporting domain** (a business subject), not per table. Read-only. No Eloquent models — use the query builder and return DTOs, arrays, or `stdClass` collections.

**Naming:** suffix with `AnalyticsRepository`. Name the subject, not a table.

```
RevenueAnalyticsRepository
UserEngagementAnalyticsRepository
```

**Methods** are named after the question they answer:

```php
class RevenueAnalyticsRepository
{
    public function monthlyRecurringRevenue(DateRange $range): Collection;
    public function revenueByRegion(DateRange $range, ?int $teamId = null): Collection;
    public function topCustomersByLifetimeValue(int $limit = 10): Collection;
}
```

```php
public function revenueByRegion(DateRange $range): Collection
{
    return DB::table('orders')
        ->join('customers', 'customers.id', '=', 'orders.customer_id')
        ->join('regions', 'regions.id', '=', 'customers.region_id')
        ->whereBetween('orders.created_at', [$range->start, $range->end])
        ->groupBy('regions.name')
        ->selectRaw('regions.name as region, SUM(orders.total) as revenue')
        ->get();
}
```

**Promote to a Query object** when a query takes 3+ params or grows conditional branches. One class, one `run(...)` method:

```php
class CohortRetentionQuery
{
    public function run(DateRange $range, string $granularity, int $cohortSize): Collection;
}
```

---

## 3. Rules of thumb

| | Transactional | Analytics |
|---|---|---|
| Unit | One per model | One per reporting domain |
| Returns | Eloquent models | DTOs / arrays / `stdClass` |
| Backing | Single table (ORM) | Joined tables (query builder) |
| Mutations | Yes | No — read-only |

- Analytics repos never mutate. Transactional repos never do reporting joins.
- Point heavy analytics at a read replica or a materialized/reporting table when possible.
- Controllers and services talk to repos — never `DB::` or Eloquent directly outside a repo.

---

## 4. Enforcing it

Wire into CI behind one `composer check` script: **PHPStan** (level 6+) for return-type contracts, **Deptrac** to forbid `DB::`/Eloquent outside repos, and **PHPArkitect** for the naming rules if drift becomes a problem.
