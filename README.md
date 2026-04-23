# hexagonal-transaction-manager-in-go

## Introduction

In many real‑world systems, a single business operation often involves multiple data modifications. For example, placing
an order may require:

- creating an order record
- updating inventory
- charging a payment
- recording a financial transaction

All of these steps must succeed or fail together. If one step fails while others succeed, the system may end up in an
inconsistent state.

This is exactly the problem **transactions** are designed to solve.

A database transaction guarantees **atomicity**: either all operations inside the transaction are committed, or none of
them are.

In traditional layered architectures, transaction management is often handled inside repositories or through framework
features such as annotations or middleware.

However, things become less straightforward when using **Hexagonal Architecture (Ports and Adapters)**.

Hexagonal Architecture promotes strict separation between application logic and infrastructure concerns. Business use
cases should not depend on databases, frameworks, or technical details.

This raises an important architectural question:

### Where should transaction management live in a hexagonal system?

If transactions are implemented inside repositories, coordinating multiple repository operations within a single atomic
unit becomes difficult. On the other hand, implementing transactions directly inside use cases couples business logic to
infrastructure concerns such as database drivers or ORM frameworks.

---

## Example

### Trade Repository

```go
type TradeRepository struct {
db *sql.DB
}

func (r *TradeRepository) Save(ctx context.Context, trade Trade) error {
tx, err := r.db.BeginTx(ctx, nil)
if err != nil {
return err
}

_, err = tx.ExecContext(ctx,
"INSERT INTO trades (symbol, qty) VALUES (?, ?)",
trade.Symbol,
trade.Qty,
)
if err != nil {
tx.Rollback()
return err
}

return tx.Commit()
}
````

### Balance Repository

````go
type BalanceRepository struct {
db *sql.DB
}

func (r *BalanceRepository) Update(ctx context.Context, userID int, amount int) error {
tx, err := r.db.BeginTx(ctx, nil)
if err != nil {
return err
}

_, err = tx.ExecContext(ctx,
"UPDATE balances SET amount = amount - ? WHERE user_id = ?",
amount,
userID,
)
if err != nil {
tx.Rollback()
return err
}

return tx.Commit()
}
````

### Problems with this approach

- Too much duplicated code
- The repository becomes bloated in terms of responsibilities
- Applying a small change (for example, the isolation level) requires modifying all repositories
- It violates the Single Responsibility Principle
- The repository ends up acting as both a transaction handler and a data access component

### No Shared Transaction

Repositories also cannot operate within a shared transaction.
If each repository creates its own transaction:

- Repositories become isolated
- There is no way to share a single transaction across multiple repositories
- Coordinating business rules becomes impossible
- Repository is tied to the infrastructure.

## Using Transactions Inside the Use Case

Another approach is to manage the transaction directly inside the use case.

```go
func (u *BuyUseCase) Buy(ctx context.Context, userID int, trade Trade) error {
tx, err := u.db.BeginTx(ctx, nil)
if err != nil {
return err
}

_, err = tx.ExecContext(ctx,
"INSERT INTO trades (symbol, qty) VALUES (?, ?)",
trade.Symbol,
trade.Qty,
)
if err != nil {
tx.Rollback()
return err
}

_, err = tx.ExecContext(ctx,
"UPDATE balances SET amount = amount - ? WHERE user_id = ?",
trade.Price,
userID,
)
if err != nil {
tx.Rollback()
return err
}

return tx.Commit()
}
````

However, this approach introduces another set of problems:

- The Use Case becomes dependent on sql.DB
- Business logic becomes coupled to the database driver
- Testing becomes harder
- It introduces dependency on a specific ORM

- This means the dependency rule in Hexagonal Architecture is violated.

This article explores a clean solution to this problem:
mplementing a **Transaction Manager** as a Port in Hexagonal Architecture. Using a real Go implementation as an example,
we will see how transaction boundaries can be managed cleanly while keeping application logic independent from
infrastructure.

## The Hexagonal Solution: Transaction Manager as a Port

Hexagonal Architecture (Ports & Adapters) provides a clean way to isolate
business logic from infrastructure details. In this style, the application
layer depends only on **abstractions**, not concrete implementations.

To properly support transactions in a hexagonal system, we introduce a new
application-level port: the **Transaction Manager**.

### What is a Port?

A Port is an *abstraction* defined by the application layer that expresses a
capability the domain or use cases need — without saying *how* it is implemented.

Examples:

- the domain needs to “store a trade” → `TradeRepository`
- the use case needs to “publish an event” → `EventPublisher`
- the application needs to “run code inside a transaction” → `TxManager`

### Why Transaction Manager Should Be a Port

Use cases *must* be able to express the requirement:

> "I need these operations to run atomically."

But they should not know:

- how the database driver starts a transaction
- how commits or rollbacks work
- which DB engine is used
- whether the system uses `*sql.DB`, `*sql.Tx`, GORM, Mongo, or anything else

Therefore, the ability to “run a function inside a transaction” must be modeled
as a **Port**.

In your project, that abstraction is:

```go
type TxManager interface {
WithTransaction(ctx context.Context, fn func (ctx context.Context) error) error
}
````

The use case only depends on this interface — not on any database logic.
This interface exposes one single high‑level capability:
> “Run this block of logic atomically.”

The application layer should not know how a transaction begins, commits, or rolls back.

It should only describe the business logic that must succeed or fail together.

### Why a callback?

Because it gives the transaction manager full control over the transaction boundary.

Instead of allowing the use case to manually Begin/Commit/Rollback, we provide a single safe entry point.

### Why does the callback receive a context?

The infrastructure layer may attach the active *sql.Tx to that context.

Repositories then read the same transaction out of the context, ensuring every operation inside the callback uses the
same DB session.

When there is no transaction → *sql.DB is used

When we are in WithTransaction → *sql.Tx is used

Both of these play the role of a **database session**.

## Implementing WithTransaction in the Infrastructure Layer

Now that the port is defined, we show its concrete implementation, which belongs entirely to the infrastructure layer
because it depends on *database/sql*.

````go
var txKey struct{}

type TxManager struct {
db *sql.DB
}

func NewTxManager(db *sql.DB) *TxManager {
return &TxManager{db: db}
}

func (m *TxManager) WithTransaction(ctx context.Context, fn func (ctx context.Context) error) (err error) {
tx, err := m.db.BeginTx()
if err != nil {
return err
}

defer func () {
if err != nil {
tx.Rollback()
} else {
err = tx.Commit()
}
}()

ctx = inject(ctx, tx)
err = fn(ctx)
return
}

func inject(ctx context.Context, tx *sql.Tx) context.Context {
return context.WithValue(ctx, txKey, tx)
}

````
## Why TxManager Is Not an Adapter
A common misunderstanding is to place the Transaction Manager inside the
adapter/out layer. In Hexagonal, however, an adapter is specifically
responsible for integrating the application with a **third‑party external
system**:

- external databases
- message brokers
- HTTP clients
- gRPC clients
- other microservices

Adapters translate between outside protocols and inside domain models.
Your Transaction Manager does none of these things.
It is:

- tightly coupled to the internal database driver
- not a boundary to the outside world
- not a translation layer
- not a protocol implementation
- not third‑party integration

Therefore it is not an adapter.

It is an internal infrastructure mechanism — and the correct place for its
implementation is the:

> infrastructure/

## The DBTX Bridge

Repositories still need a way to execute database queries. However, they should work with either:

- normal database connection (*sql.DB)
- an active transaction (*sql.Tx)

Both types expose similar methods such as ExecContext, QueryContext, and QueryRowContext. To avoid coupling repositories to concrete types, we introduce a small abstraction called DBTX.
````go
type DBTX interface {
    ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error)
    QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error)
    QueryRowContext(ctx context.Context, query string, args ...any) *sql.Row
}
````
Both *sql.DB and *sql.Tx satisfy this interface.

This means repositories can depend only on DBTX, allowing them to operate correctly regardless of whether a transaction is currently active.

Repositories simply execute queries using the provided DBTX without needing to know if they are inside a transaction.

## Injecting the Transaction Through Context

The remaining challenge is ensuring that repositories receive the correct database handle.

When a transaction starts, the transaction manager stores the *sql.Tx inside the request context. A small helper function then retrieves the correct database handle when repositories need it.

Conceptually the flow works like this:

- Transaction manager starts sql.Tx
- The transaction is stored inside the context
- The use case callback runs
- Repositories retrieve the correct DB object from the context
- Queries execute using either *sql.Tx or *sql.DB

A simplified helper might look like this:

````go
func GetDB(ctx context.Context, db *sql.DB) port.DBTX {
if tx, ok := ctx.Value(txKey).(*sql.Tx); ok {
return tx
}
return db
}
````
Repositories can then obtain the correct database object without knowing anything about transactions.



## The Full Transaction Flow

With all pieces in place, the runtime flow becomes very clean.
A use case coordinates the operation:

````go
func (u *MultiBuyUseCase) Execute(ctx context.Context, req Request) error {
    return u.txManager.WithTransaction(ctx, func(ctx context.Context) error {

        if err := u.tradeRepo.Create(ctx, req.Trade); err != nil {
            return err
        }

        if err := u.balanceRepo.Update(ctx, req.UserID, req.Amount); err != nil {
            return err
        }

        return nil
    })
}
````

Execution flow:

````
UseCase
   ↓
TxManager.WithTransaction
   ↓
Begin SQL Transaction
   ↓
Inject *sql.Tx into context
   ↓
Run use case callback
   ↓
Repositories read DBTX from context
   ↓
Queries execute using the same transaction
   ↓
Commit or Rollback

````
The use case remains completely unaware of database details while still controlling the transactional boundary.

## Benefits of This Approach
This architecture solves several problems at once.

First, it removes duplicated transaction logic from repositories. Transaction handling becomes centralized and consistent.

Second, it prevents infrastructure details from leaking into the application layer. Use cases no longer depend on sql.DB, sql.Tx, or any ORM.

Third, repositories become simpler. They only execute queries using the provided DBTX interface.

Fourth, use cases can safely coordinate multiple repositories within a single atomic operation.

Finally, this approach maintains the dependency rule of Hexagonal Architecture: inner layers define abstractions, outer layers provide implementations.

