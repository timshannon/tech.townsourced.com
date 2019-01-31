---
title: "BoltDB vs Badger: A Comparison of Go Key-Value databases"
draft: false
date: "2019-01-29 19:36:00"
categories: 
  - Development
tags: 
  - Development
  - Go
  - programming
  - project
  - database
keywords:
  - go
  - golang
  - development
  - programming
  - projects
  - boltdb
  - bolt
  - badger
  - badgerdb
  - bolthold
  - badgerhold
  - KV store
  - comparison
  - nosql
  - no-sql
  - database
  - embedded
  - boltdb vs badger
  - lmdb
---
When I first started working on [BoltHold](https://github.com/timshannon/bolthold) (*a simple querying and index engine
that sits on top of BoltDB*), [Badger](https://github.com/dgraph-io/badger) didn't yet exist, and 
[BoltDB](https://github.com/etcd-io/bbolt) was the clear leader of the pack for key-value, pure-go, embeddable databases.

Then Badger was released, and it was shown to be more than just a pure-go version of LSM-tree stores like RocksDB / 
LevelDB, it actually was *faster* than RocksDB.  **[Much faster](https://blog.dgraph.io/post/badger/)**.  I knew I 
wanted to build something with Badger in the future, and when an issue was opened to 
[add Badger support to Bolthold](https://github.com/timshannon/bolthold/issues/41), I jumped on it.


The first thing I had to consider in adding Badger support was whether I wanted to add Badger as another dependency and allow
users to switch between the two via a configuration flag, or build an entirely separate library.  I ended up following
the Go proverb:

> A little copying is better than a little dependency

I decided to make BadgerHold a separate library, and now that I have 88% test coverage using the exact same tests as BoltHold,
I believe that I made the right choice.  

In implementing the exact same library on two different KV stores, I like to think I have gained some perspective on 
comparing these two databases.  Dgraph have already done an 
[excellent comparison of performance](https://blog.dgraph.io/post/badger-lmdb-boltdb/) between these two libraries. This
post instead will focus on comparing criteria *other* than performance.

# Superficial Differences
When I first approached writing BadgerHold, I was already familiar with BoltDB and had used it on several other projects.
Unfortunately, this meant that the onus was on Badger to meet or conflict with my expectations upon first using the library.
With that in mind, I found the following differences to be superficial, and not a major impact in my usage of both libraries.
That being said, I felt I should list them in case they prove to be more important to you.

## File Handling 
With BoltDB, you open a database by specifying a file, and passing in some options:

```go
db, err := bolt.Open("my.db", 0600, options)
```

Badger, on the other hand, requires you to specify a multiple folders as part of an options struct instead of a single file:
```go
opts := badger.DefaultOptions
opts.Dir = "/tmp/badger"
opts.ValueDir = "/tmp/badger"
db, err := badger.Open(opts)
```
This is due to the nature of LSM-tree databases, where multiple levels are stored across multiple files.  I'm hard
pressed to come up with a *legitimate* reason why a single file would matter over a directory of files, but I must admit,
*with all other things being equal*, I'd prefer to manage an embedded database in a single file.

## Unexpected API Choices
In general I found BoltDB to fall more in line with what I would expect from a Go API compared to Badger. My expectations
being defined by those that I am used to seeing in the Go standard library.

For example, a common way to handle options in the Go standard library is to default values if not specified (see 
[http.Server](https://golang.org/pkg/net/http/#Server), and [sql.Conn](https://golang.org/pkg/database/sql/#Conn.BeginTx)).

Bolt's `Open` method will accept a nil option and open up the data file with defaults.

```go
db, err := bolt.Open("my.db", 0600, nil)
```

Badger's option argument doesn't accept a pointer, so a struct needs to be passed in.  If you want to use the default
options, you need to make a copy of an exported struct that contains defaults.  Furthermore, you'll *always* need to
change those defaults because the data directories are stored in the options type.

```go
opts := badger.DefaultOptions
opts.Dir = "/tmp/badger"
opts.ValueDir = "/tmp/badger"
db, err := badger.Open(opts)
```

In the standard Go library, constants and enumerations are usually stored within the same package where they are used,
or local enumerations defined from external packages exported enumerations (see 
[gzip](https://golang.org/pkg/compress/gzip/#pkg-constants) and [os](https://golang.org/pkg/os/#pkg-constants)).

Badger requires you to import the [badger/options](https://godoc.org/github.com/dgraph-io/badger/options) package to 
get the enumerations for `FileLoadingMode`.

# Behavior Differences
The following are more significant differences between how BoltDB and Badger behave.

## Buckets
BoltDB has a concept of [buckets](https://godoc.org/github.com/etcd-io/bbolt#Bucket) in a single database. Comparable
to tables in a relational database, this allows you to store different types of data in the same database, and not 
worry about row conflicts.

In BoltHold, I used different buckets to store different types.  Since Badger doesn't have similar functionality, I
had to replicate it by prefixing keys with the type being stored, after which I could use Badger's 
[ValidForPrefix](https://godoc.org/github.com/dgraph-io/badger#Iterator.ValidForPrefix) method when iterating to make
sure queries only apply to the proper type. This allows the behavior between BoltHold and BadgerHold to be the same when 
storing different types in the same database. BoltDB most likely does something similar to create it's bucket functionality,
but it's nice not to have to re-implement it.

## Iterators
BoltDB allows you to create a number of Cursors on any given transaction.  Badger, on the other hand, only allows  one 
iterator at a time in Read / Write transactions.

In BoltHold and BadgerHold, you have the ability to write subqueries, which allow you to filter records based on another 
query from within that same transaction.

```go
bh.Where("Name").MatchFunc(func(ra *bh.RecordAccess) (bool, error) {
	// find where name exists in more than one category
	record, ok := ra.Record().(*ItemTest)
	if !ok {
		return false, fmt.Errorf("Record is not ItemTest, it's a %T", ra.Record())
	}

	var result []ItemTest

	err := ra.SubQuery(&result,
		bh.Where("Name").Eq(record.Name).And("Category").Ne(record.Category))
	if err != nil {
		return false, err
	}

	if len(result) > 0 {
		return true, nil
	}

	return false, nil
})
```
This type of query is much harder to implement in Badger for `Update` and `Delete` queries than in BoltDB due to this
single iterator limit from within R/W transactions.

## Memory Usage
The single most surprising difference for me between BoltDB and Badger, was the disparity in memory usage.  In my testing
I found Badger to use *significantly* more memory than BoltDB.  So much so that  I had to rewrite my BadgerHold tests from using a 
separate database for each test to using a single shared database for all tests.  I couldn't complete the entire suite
of tests on my 8GB VM due to out of memory errors. Even after changing all of Badger's options to their
[lowest memory settings](https://github.com/dgraph-io/badger#memory-usage), I was unable to get the full suite to finish.

To be fair, this is most likely by design, and the source of a lot of the impressive performance differences between BoltDB
and Badger. The memory usage was also exasperated by opening multiple databases, which meant there needed to be enough 
memory for each of those database's buffer cache. There are many scenarios where this trade off of memory for speed is 
well worth the cost, however, I was personally surprised by the extent of it, as I hadn't seen it mentioned in any of 
the documentation or blog posts.

# Summary

| *Criteria*                       		| BoltDB                	| Badger	                	|
| ----------                      		| -------------         	| -------------                		|
| [File Handling](#file-handling)   		| Single File	          	| Multiple Directories	         	| 
| [API](#unexpected-api-choices)    		| Idiomatic			| Unexpected				|
| [Bucket Support](#buckets)		    	| Yes				| No					|
| [Iterators](#iterators)			| No Restrictions		| One R/W iterator at a time		|
| [Memory Usage](#memory-usage)			| Low				| High					|

The biggest differences between BoltDB and Badger, stem from their underlying technologies, and not their implementation
details.  If you are picking one over the other based solely on performance metrics, I would closely consider your
usage case instead.

My recommendation would be to use BoltDB for scenarios where you are more memory constrained or where the performance
gain isn't worth the cost in memory usage.  Examples of this may be projects on mobile phones, tablets, Raspberry 
Pi-like devices or command line applications where you need some persistent storage.

For micro-services and projects you'd tend to run on a server where more memory is available, or projects where you expect
a lot of writes like for logging, I would recommend Badger.

I would also recommend using one shared Badger database, rather then opening multiple separate databases to limit your
memory usage.  Using a library like BadgerHold, or using similar key-prefixing methods in your own code, should allow
you to manage your data types separately within the same database.

If you choose to use [BoltHold](https://github.com/timshannon/bolthold), or 
[BadgerHold](https://github.com/timshannon/badgerhold), I would recommend aliasing the package import as `bh` and since
both library's APIs are nearly identical, you could easily switch your project from one to another if you
find that one library will work better for you.

```go
import (
	bh "github.com/timshannon/badgerhold"
)

// this code will run the same on BoltHold and BadgerHold

store.UpdateMatching(&Person{}, bh.Where("Death").Lt(bh.Field("Birth")), func(record interface{}) error {
	update, ok := record.(*Person)
	if !ok {
		return fmt.Errorf("Record isn't the correct type!  Wanted Person, got %T", record)
	}

	update.Birth, update.Death = update.Death, update.Birth

	return nil
})
```

