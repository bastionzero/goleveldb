This is BastionZero's fork of a [LevelDB key/value database](https://github.com/google/leveldb) implementation in the [Go programming language](https://go.dev).

Why we forked
-----------
The nodejs [library](https://www.npmjs.com/package/classic-level) we use in the [ZLI](https://github.com/bastionzero/zli) is a direct wrapper for the original leveldb c++ code. That code uses fnctl locks to secure access to the db, while the golang library we [forked](https://github.com/syndtr/goleveldb) uses flocks. On linux, flocks have an independent implementation from fnctl locks which means the two are incompatible whereas on other systems, flocks are implemented using fnctl locks.

The most important difference between the two (for us) is that fnctl locks map processes/pids to files. That means that a single lock is acquired for a single pid to access a given file. It does not allow for multi-threaded single processes which would use the same pid but otherwise uncoordinated access attempts within the code. Flocks maps a file handler to a file. This does allow for multi-threading because each thread would have a distinct file handler open to the file.

This change is not more or less correct than using a flock, all it does it make it more compatible with other leveldb libraries that are implemented as bindings of the original c++ code, but there are benefits/detriments to doing it either way.

[Here](https://lwn.net/Articles/586904/) is a good resource on locks

Installation
-----------

	go get github.com/bastionzero/goleveldb/leveldb

For a drop-in replacement, insert this line into your go.mod file:
 ```go
replace github.com/syndtr/goleveldb v1.0.0 => github.com/bastionzero/goleveldb v1.0.0
```

Requirements
-----------

* Need at least `go1.14` or newer.

Usage
-----------

Create or open a database:
```go
// The returned DB instance is safe for concurrent use. Which mean that all
// DB's methods may be called concurrently from multiple goroutine.
db, err := leveldb.OpenFile("path/to/db", nil)
...
defer db.Close()
...
```
Read or modify the database content:
```go
// Remember that the contents of the returned slice should not be modified.
data, err := db.Get([]byte("key"), nil)
...
err = db.Put([]byte("key"), []byte("value"), nil)
...
err = db.Delete([]byte("key"), nil)
...
```

Iterate over database content:
```go
iter := db.NewIterator(nil, nil)
for iter.Next() {
	// Remember that the contents of the returned slice should not be modified, and
	// only valid until the next call to Next.
	key := iter.Key()
	value := iter.Value()
	...
}
iter.Release()
err = iter.Error()
...
```
Seek-then-Iterate:
```go
iter := db.NewIterator(nil, nil)
for ok := iter.Seek(key); ok; ok = iter.Next() {
	// Use key/value.
	...
}
iter.Release()
err = iter.Error()
...
```
Iterate over subset of database content:
```go
iter := db.NewIterator(&util.Range{Start: []byte("foo"), Limit: []byte("xoo")}, nil)
for iter.Next() {
	// Use key/value.
	...
}
iter.Release()
err = iter.Error()
...
```
Iterate over subset of database content with a particular prefix:
```go
iter := db.NewIterator(util.BytesPrefix([]byte("foo-")), nil)
for iter.Next() {
	// Use key/value.
	...
}
iter.Release()
err = iter.Error()
...
```
Batch writes:
```go
batch := new(leveldb.Batch)
batch.Put([]byte("foo"), []byte("value"))
batch.Put([]byte("bar"), []byte("another value"))
batch.Delete([]byte("baz"))
err = db.Write(batch, nil)
...
```
Use bloom filter:
```go
o := &opt.Options{
	Filter: filter.NewBloomFilter(10),
}
db, err := leveldb.OpenFile("path/to/db", o)
...
defer db.Close()
...
```
Documentation
-----------

You can read package documentation [here](https://pkg.go.dev/github.com/bastionzero/goleveldb).
