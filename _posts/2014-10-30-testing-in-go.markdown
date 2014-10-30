---
layout: post
title:  "Testing in Go: testing package, mocking and interfaces"
date:   2014-10-30 19:45:00
categories: blog golang
---

# Testing in Go

Testing in Go is pretty straightforward. The standard library [`testing`](http://golang.org/pkg/testing) is extremely powerful and there is really little need for additional testing frameworks.
In production I use tend to go with `testing` package for most of my own packages, then the [`gocheck`](https://labix.org/gocheck) library when testing integrations.
I just wanted to take a second to demonstrate how I got about testing Go applications. As well as explain a few tips and tricks for designing your go software, so that is easy to test.

## Keep it simple, use the `testing` standard library
To start with I want to look what [@bmizerany](https://twitter.com/bmizerany) talked about at this years first [dotGo](http://dotgo.io) conference in Paris.
When he first started with Go, he felt like something was missing. Coming from the world of Ruby, he noticed he didn’t have all those lovely testing helper methods you get with RSpec.
Things like one line checking statements, regular expressions asserters and magic mocking functions/objects for stubbing out behaviour.

So as an example, I am going to demonstrate some of the nice little helpers in the `gocheck` package. So you can see the kind of thing he was missing.

{% highlight go %}
c.Check(year, gocheck.Matches, "[0-9]{4}")
{% endhighlight %}

But how much code is this actually replacing?

{% highlight go %}
if ok, _ := regexp.StringMatch("[0-9]{4}", year); !ok {
    t.Errorf("Expected to match /[0-9]{4}/, instead found %s", year)
}
{% endhighlight %}

The only real extra work you have to do with the `testing` package, is format your own error response.
Which, with a few template error strings, you can get this done quickly.
Not worth importing around 4000+ lines of code for.
In fact you even need a lot more boiler plate code to set up a `gocheck` suite.
Where as with `testing` you just write an exported function, that begins with the word `Test` within your testing package.

> I am being a little mean here, I actually like gocheck. However, I will explain what I think it's good for further down in this post.

Blake actually started out by writing his own assertion package for go [bmizerany/assert](http://github.com/bmizerany/assert).
Only to come to the conclusion that all he needed to do was "learn to write an if-statement and t.Errorf".
In order to exclude including his assertion package in to all of his tests.

## Stubbing my foot and making a mockery of myself

So what about mocking? I want to take you through a discovering I made, while testing interactions with databases.
So first take a look at `database/sql` package. If you're not familiar with this package, it is how most go programs interface with an sql database.
It is a fairly generic package, where you include other driver packages, which magically interface with it. For example, `lib/pq` driver for PostgreSQL.
However, this posed a problem for me. I found myself getting sick of having to interact with the database to get full test coverage of my program.
I wanted to answer questions like:

- How to I get this database result to return an error when I am scanning through it?
- How do I get this transaction to fail when preparing a statement.
- How do I tell whether my program has called `tx.Rollback()` when and error occurs within the transaction?

I was testing my packages protocols for dealing with database issues. So really, I need a way to mock the structs within `database/sql`.
However, these are statically typed structs. There is no amount Ruby-like meta voodoo that is going to save me here.

So how do I solve this? The important point to note is that I'm testing my programs behaviour and protocol when interacting with components of the `database/sql`.
Integration tests here, didn't really solve this problem. Sure with a bit of coercion I could get the database to trigger errors in certain places. However,
integration tests are only really useful for testing the behaviour of the database, under the conditions and parameters you supply. For example, that your query behaves
as expected. Do you get back the right number of results. Are they in the format expected? Not does my program rollback a transaction when it fails to process a response.

### Interfaces to the rescue!

So my initial solution was interfaces! Big interfaces! To actually interface the entire of the `sql.DB` struct and the `sql.Tx` structs.
Then get all my database interacting functions to take these interfaces, instead of the sql structs.

> I was actually writing a lot of this on the train to dotGo. Only to scrap it after listening to the Go team.

What did I gain from this? Well initially quite a lot.
By defining an interface for both the database and a transaction, due to Golangs lovely type system, `database/sql` just clicked in to place nicely.
Then all I need to do was make dummy implementations of database for test cases. I could make them in my test packages. Then I would be able to ask the questions I needed.

- Did that transaction have rollback called on it, when `x` return an error?
- Did my logger print out the expected error when `db.Begin` was called?
- Did my processing package hand `tx.Query` the sql statement I expected?

So what did this cost me? I needed to write massive fake implementations of `sql.DB` and `sql.Tx` in order to just check whether a transaction was created.
To check whether rollback was called. I needed to implement all the methods of the interface. Plus in many cases, these new concrete test types were inflexible.
Sure this transaction return an error on `tx.Query`. But what happened when I needed that error to be nil?
Either I needed to have a large amount of customisable options on my new test implementation, or I needed multiple implementations.

### Getting some closure

But then I had a clever thought (well I thought it was clever at the time). An idea for a `MockDB` and a `MockTx` that was flexible enough to be my database swiss army knife.
The idea was simple.
These types would implement my interfaces by delegating to an anonymous function that is stored within the struct as an exported field, with a similar name to the function you were calling.

The following is an example of what I managed to create.

{% highlight go %}
// MockTx implements Tx.
//
// It is composed of fields to be used directly as return
// values of the functions which match the Tx interface.
type MockTx struct {
	Commitf   func() error
	Execf     func(query string, args ...interface{}) (sql.Result, error)
	Preparef  func(query string) (*sql.Stmt, error)
	Queryf    func(query string, args ...interface{}) (*sql.Rows, error)
	QueryRowf func(query string, args ...interface{}) *sql.Row
	Rollbackf func() error
	Stmtf     func(stmt *sql.Stmt) *sql.Stmt
}

func (m *MockTx) Commit() error                                       { return m.Commitf() }
func (m *MockTx) Exec(q string, a ...interface{}) (sql.Result, error) { return m.Execf(q, a) }
func (m *MockTx) Prepare(q string) (*sql.Stmt, error)                 { return m.Preparef(q) }
func (m *MockTx) Query(q string, a ...interface{}) (*sql.Rows, error) { return m.Queryf(q, a) }
func (m *MockTx) QueryRow(q string, a ...interface{}) *sql.Row        { return m.QueryRowf(q, a) }
func (m *MockTx) Rollback() error                                     { return m.Rollbackf() }
func (m *MockTx) Stmt(s *sql.Stmt) *sql.Stmt                          { return m.Stmt(s) }
{% endhighlight %}

Now this was great.
I got to use the power of anonymous closures to stub out the functionality of database and transaction.
However, I had introduced abstraction with the sole intention of making testing easier.
It has, in no way benefited the concrete solution to the problem I was originally trying to solve. Inserting things in to the database. Or reporting on the contents of a database.

### Setting myself up for bad practice

Interfaces in Go represent sets of concrete types.
This allows for small interfaces to be described. Which then capture large groups of concrete types.
These should be used to protect interactions between other types.
So that functions have access only to the minimal interface required to serve that functions purpose.
Functions which have large concerns over interaction, can do so by composing interface types. Go makes interface type composition easy.

My colleague [e-dard](http://github.com/e-dard) interjects here and gives me the solution I need. Smaller interfaces comprising of database/transaction types methods.
Things like the following.

{% highlight go %}
type Execer interface {
    Exec(query string, args ...interface{}) (sql.Result, error)
}

type Committer interface {
    Commit() error
}

type ExecCommitter interface {
    Execer
    Committer
}
{% endhighlight %}

Define types for the functions that operate over them. One of the key points made by the panel at dotGo is that interfaces are intended to be used as function arguments.
Not as return types. So write functions which just take the `Querier` interface, or the `QueryRollbackCommitter` composed interface.
Let functions construct concrete responses to the problems they are solving. Yet provided them with the minimum information they need to produce these results, via these defensive interfaces.
These interfaces then allow for simple mocking, with the intention of testing interaction and protocol.

### Mocking these smaller interfaces

#### 1. Write a load of types, which allow for inspection and stubbing.

Write a set of concrete types, which implement these interfaces.
Extend these types with functionality to stub out response, or capture input.
Here is an example:

{% highlight go %}
type DummyExecer struct {
    query string
    args []interface{}
    Result sql.Result
    Error error
}

func (d *DummyExecer) Exec(q string, a ...interface{}) (sql.Result, error) {
    d.query = q
    d.args = a
    return d.Result, d.Error
}
{% endhighlight %}

An instance of this `DummyExecer` type can be constructed with the desired response from `Exec(...)`.
Then passed to a function, which operates over types of `Execer`.
Then once that function is complete, inspection of the arguments passed to `Exec` can be made via the fields `query` and `args`.

#### 2. Mocking with anonymous functions in fields

Much like my example of a `MockTx`, create types which contain fields of functions with the same type signatures as those in the interface.
Then delegate responsibility to those within the concrete methods of the struct.
For example:
{% highlight go %}
type MockCommitter struct {
    Commitf func() error
}

func (m MockCommitter) Commit() error { return m.Commitf() }
{% endhighlight %}

This allows for the creation of types, which can compose anonymous closures on the fly.
This is really flexible when it comes to quickly stubbing out arguments to functions.

{% highlight go %}
commitTransaction(MockCommitter{
    Commitf: func() error {
        return errors.New("Something went wrong on commit!")
    }
})
{% endhighlight %}

#### 3. Make functions that comply with your interfaces! (personal new favourite for small interfaces)

I like this one. Works great for small interfaces.
Functions can take receiver arguments to named types that aren't interface types. Since interfaces can't have associated implementation.
Because of this, we can name things like functions, slices and maps. Then associate functions with them, so they comply with interfaces.
One of the best examples of this being used is within the `net/http` package. Between the `Handler` interface and the `HandlerFun` function type.
The code looks something like this:
{% highlight go %}
type Handler struct {
    ServeHttp(rw ResponseWriter, req *Request)
}

type HandlerFunc func(rw ResponseWriter, req *Request)

func (h HandlerFunc) ServeHttp(rw ResponseWriter, req Request) {
    h(rw, req)
}
{% endhighlight %}

This is amazingly powerful, because it allows us to construct concrete implementations of interfaces, out of anonymous function/closures.
Much like, I suppose, single function interfaces in Java 8, which can be implemented using lambdas of the same signature.
However, in Go you can assign further functions to named function types. So that they comply with larger interfaces.

So taking our example of the `Execer` interface, we could create something like the following.

{% highlight go %}
type ExecerFunc func(string, ...interface{}) (sql.Result, error)

func (e ExecerFunc) Exec(q string, a ...interface{}) (sql.Result, error) {
    return e(q, a...)
}

func TestPersister(t *testing.T) {
    var (
        query string
        args []interface{}
    )

    exec := ExecerFunc(func(q string, a ...interface{}) (sql.Result, error){
        query = q
        args = a
        return nil, nil
    })

    expquery := "select * from $1;"
    persist.Persist(exec, expquery, "blah")

    if query != expquery {
        t.Errorf("Expected '%s', go '%s'", expquery, query)
    }

    if args[0] != "blah" {
        t.Errorf("Expected 'blah', got '%s'", args[0])
    }
}
{% endhighlight %}

I don't know why I like the final example the most. I think its the conciseness of constructing closures into types which comply with interfaces.

### Integrations and some extra tips

So I haven't done testing frameworks any justice.
So far I have painted them as bulky addons, with little extra functionality and with the overhead of required extra boilerplate code.
So I want to look at `gocheck` and its positives.
Truthfully, `gocheck` isn't for the sole purpose of bundling in a load of one line checkers. Which produce nicely formatted failure messages.
It also allows you to define your own set of checkers. Which can be nice to share between your Go projects.
Most importantly, it allows for the orchestration of test set-up/tear-down at a suite level (you can have many test suites in one package) and an individual test function level.

This is a really intuitive and tidy way of organising tests, particularly when interacting with a database.
I like to put all of my database connection and set up within a suite `SetUpSuite` function.
Add some database fixtures in to a suites `SetUpTest` function.
Then truncate all my database tables with the suites `TearDownTest` function.
This allows for easy isolation of each test case from one another, when they all interact on the database. The following example illustrates a `gocheck.Suite`.
{% highlight go %}
var _ = gocheck.Suite(&TestSuite{})

func Test(t *testing.T) ( gocheck.TestingT(t) )

type TestSuite struct {
    db *sql.DB
}

func (t *TestSuite) SetUpSuite(c *gocheck.C) {
    var err error
    t.db, err = sql.Connect("blah connection string")
    if err != nil {
        panic(err)
    }
}

func (t *TestSuite) SetUpTest(c *gocheck.C) {
    runFixtures(t.db, "results.sql")
}

func (t *TestSuite) TearDownTest(c *gocheck.C) {
    truncateTables(t.db, "results")
}
{% endhighlight %}

## Conclusions (and TL;DR)

- Keep interfaces small.
- Use interfaces as arguments, not return values.
- Make use of closures with mocking structs or named function types, when testing interactions and protocol.
- Use `testing` package as much as possible!
- Try out `gocheck` for orchestrating integration tests.
