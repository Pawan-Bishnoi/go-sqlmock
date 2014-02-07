# Sql driver mock for Golang

This is a **mock** driver as **database/sql/driver** which is very flexible and pragmatic to
manage and mock expected queries. All the expectations should be met and all queries and actions
triggered should be mocked in order to pass a test. See exported [api on godoc](http://godoc.org/github.com/DATA-DOG/go-sqlmock)

## Install

    go get github.com/DATA-DOG/go-sqlmock

## Use it with pleasure

An example of some database interaction which you may want to test:

``` go
package main

import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql"
    "github.com/kisielk/sqlstruct"
    "fmt"
    "log"
)

const ORDER_PENDING = 0
const ORDER_CANCELLED = 1

type User struct {
    Id int `sql:"id"`
    Username string `sql:"username"`
    Balance float64 `sql:"balance"`
}

type Order struct {
    Id int `sql:"id"`
    Value float64 `sql:"value"`
    ReservedFee float64 `sql:"reserved_fee"`
    Status int `sql:"status"`
}

func cancelOrder(id int, db *sql.DB) (err error) {
    tx, err := db.Begin()
    if err != nil {
        return
    }

    var order Order
    var user User
    sql := fmt.Sprintf(`
SELECT %s, %s
FROM orders AS o
INNER JOIN users AS u ON o.buyer_id = u.id
WHERE o.id = ?
FOR UPDATE`,
    sqlstruct.ColumnsAliased(order, "o"),
    sqlstruct.ColumnsAliased(user, "u"))

    // fetch order to cancel
    rows, err := tx.Query(sql, id)
    if err != nil {
        tx.Rollback()
        return
    }

    defer rows.Close()
    // no rows, nothing to do
    if !rows.Next() {
        tx.Rollback()
        return
    }

    // read order
    err = sqlstruct.ScanAliased(&order, rows, "o")
    if err != nil {
        tx.Rollback()
        return
    }

    // ensure order status
    if order.Status != ORDER_PENDING {
        tx.Rollback()
        return
    }

    // read user
    err = sqlstruct.ScanAliased(&user, rows, "u")
    if err != nil {
        tx.Rollback()
        return
    }
    rows.Close() // manually close before other prepared statements

    // refund order value
    sql = "UPDATE users SET balance = balance + ? WHERE id = ?"
    refundStmt, err := tx.Prepare(sql)
    if err != nil {
        tx.Rollback()
        return
    }
    defer refundStmt.Close()
    _, err = refundStmt.Exec(order.Value + order.ReservedFee, user.Id)
    if err != nil {
        tx.Rollback()
        return
    }

    // update order status
    order.Status = ORDER_CANCELLED
    sql = "UPDATE orders SET status = ?, updated = NOW() WHERE id = ?"
    orderUpdStmt, err := tx.Prepare(sql)
    if err != nil {
        tx.Rollback()
        return
    }
    defer orderUpdStmt.Close()
    _, err = orderUpdStmt.Exec(order.Status, order.Id)
    if err != nil {
        tx.Rollback()
        return
    }
    return tx.Commit()
}

func main() {
    db, err := sql.Open("mysql", "root:nimda@/test")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    err = cancelOrder(1, db)
    if err != nil {
        log.Fatal(err)
    }
}
```

And the clean nice test:

``` go
package main

import (
    "database/sql"
    "github.com/DATA-DOG/go-sqlmock"
    "testing"
    "fmt"
)

// will test that order with a different status, cannot be cancelled
func TestShouldNotCancelOrderWithNonPendingStatus(t *testing.T) {
    // open database stub
    db, err := sql.Open("mock", "")
    if err != nil {
        t.Errorf("An error '%s' was not expected when opening a stub database connection", err)
    }

    // columns are prefixed with "o" since we used sqlstruct to generate them
    columns := []string{"o_id", "o_status"}
    // expect transaction begin
    sqlmock.ExpectBegin()
    // expect query to fetch order and user, match it with regexp
    sqlmock.ExpectQuery("SELECT (.+) FROM orders AS o INNER JOIN users AS u (.+) FOR UPDATE").
        WithArgs(1).
        WillReturnRows(sqlmock.RowsFromCSVString(columns, "1,1"))
    // expect transaction rollback, since order status is "cancelled"
    sqlmock.ExpectRollback()

    // run the cancel order function
    err = cancelOrder(1, db)
    if err != nil {
        t.Errorf("Expected no error, but got %s instead", err)
    }
    // db.Close() ensures that all expectations have been met
    if err = db.Close(); err != nil {
        t.Errorf("Error '%s' was not expected while closing the database", err)
    }
}

// will test order cancellation
func TestShouldRefundUserWhenOrderIsCancelled(t *testing.T) {
    // open database stub
    db, err := sql.Open("mock", "")
    if err != nil {
        t.Errorf("An error '%s' was not expected when opening a stub database connection", err)
    }

    // columns are prefixed with "o" since we used sqlstruct to generate them
    columns := []string{"o_id", "o_status", "o_value", "o_reserved_fee", "u_id", "u_balance"}
    // expect transaction begin
    sqlmock.ExpectBegin()
    // expect query to fetch order and user, match it with regexp
    sqlmock.ExpectQuery("SELECT (.+) FROM orders AS o INNER JOIN users AS u (.+) FOR UPDATE").
        WithArgs(1).
        WillReturnRows(sqlmock.RowsFromCSVString(columns, "1,0,25.75,3.25,2,10.00"))
    // expect user balance update
    sqlmock.ExpectExec("UPDATE users SET balance").
        WithArgs(25.75 + 3.25, 2). // refund amount, user id
        WillReturnResult(sqlmock.NewResult(0, 1)) // no insert id, 1 affected row
    // expect order status update
    sqlmock.ExpectExec("UPDATE orders SET status").
        WithArgs(ORDER_CANCELLED, 1). // status, id
        WillReturnResult(sqlmock.NewResult(0, 1)) // no insert id, 1 affected row
    // expect a transaction commit
    sqlmock.ExpectCommit()

    // run the cancel order function
    err = cancelOrder(1, db)
    if err != nil {
        t.Errorf("Expected no error, but got %s instead", err)
    }
    // db.Close() ensures that all expectations have been met
    if err = db.Close(); err != nil {
        t.Errorf("Error '%s' was not expected while closing the database", err)
    }
}

// will test order cancellation
func TestShouldRollbackOnError(t *testing.T) {
    // open database stub
    db, err := sql.Open("mock", "")
    if err != nil {
        t.Errorf("An error '%s' was not expected when opening a stub database connection", err)
    }

    // expect transaction begin
    sqlmock.ExpectBegin()
    // expect query to fetch order and user, match it with regexp
    sqlmock.ExpectQuery("SELECT (.+) FROM orders AS o INNER JOIN users AS u (.+) FOR UPDATE").
        WithArgs(1).
        WillReturnError(fmt.Errorf("Some error"))
    // should rollback since error was returned from query execution
    sqlmock.ExpectRollback()

    // run the cancel order function
    err = cancelOrder(1, db)
    // error should return back
    if err == nil {
        t.Error("Expected error, but got none")
    }
    // db.Close() ensures that all expectations have been met
    if err = db.Close(); err != nil {
        t.Errorf("Error '%s' was not expected while closing the database", err)
    }
}
```

## Expectations

All **Expect** methods return a **Mock** interface which allow you to describe
expectations in more details: return an error, expect specific arguments, return rows and so on.
**NOTE:** that if you call **WithArgs** on a non query based expectation, it will panic

A **Mock** interface:

``` go
type Mock interface {
	WithArgs(...driver.Value) Mock
	WillReturnError(error) Mock
	WillReturnRows(driver.Rows) Mock
	WillReturnResult(driver.Result) Mock
}
```

As an example we can expect a transaction commit and simulate an error for it:

``` go
sqlmock.ExpectCommit().WillReturnError(fmt.Errorf("Deadlock occured"))
```

In same fashion, we can expect queries to match arguments. If there are any, it must be matched.
Instead of result we can return error..

``` go
sqlmock.ExpectQuery("SELECT (.*) FROM orders").
    WithArgs("string value").
    WillReturnResult(sqlmock.NewResult(0, 1))
```

**WithArgs** expectation, compares values based on their type, for usual values like **string, float, int**
it matches the actual value. Types like **time** are compared only by type. Other types might require different ways
to compare them correctly, this may be improved.

## Run tests

    go test

## Documentation

See it on [godoc](http://godoc.org/github.com/DATA-DOG/go-sqlmock)

## TODO

- handle argument comparison more efficiently

## Contributions

Feel free to open a pull request. Note, if you wish to contribute an extension to public (exported methods or types) -
please open an issue before, to discuss whether these changes can be accepted. All backward incompatible changes are
and will be treated cautiously

## License

The [three clause BSD license](http://en.wikipedia.org/wiki/BSD_licenses)

