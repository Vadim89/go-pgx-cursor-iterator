# go-pgx-cursor-iterator
[![Actions Status](https://github.com/Vadim89/go-pgx-cursor-iterator/workflows/push/badge.svg)](https://github.com/Vadim89/go-pgx-cursor-iterator/actions)
[![Coverage Status](https://coveralls.io/repos/github/Eun/go-pgx-cursor-iterator/badge.svg?branch=master)](https://coveralls.io/github/Eun/go-pgx-cursor-iterator?branch=master)
---
A golang package for fetching big chunks of rows from a postgres database using a cursor.

```go
type User struct {
	Name string `db:"name"`
	Role string `db:"role"`
}
values := make([]User, 1000)

ctx := context.Background()
iter, err := NewCursorIterator(ctx, pool, values, time.Minute, "SELECT * FROM users WHERE role = $1", "Guest")
if err != nil {
	panic(err)
}
defer iter.Close()
for iter.Next() {
	fmt.Printf("Name: %s\n", values[iter.ValueIndex()].Name)
}
if err := iter.Error(); err != nil {
	panic(err)
}
```

## Behind the scenes
With the first `Next()` call the iterator will start a transaction and define the cursor.  
After that it will fetch the first chunk of rows.

With the following `Next()` calls the iterator will first consume the already fetched rows.  
If it iterated over all (pre)fetched rows it will fetch the next chunk and repeats the process.

With a chunk/batch size of 100 items and 250 rows in the database,
the iterator will perform 3 fetches:
1. Fetching 100 items (150 rows left)
2. Fetching 100 items (50 rows left)
3. Fetching 100 items, but will only return 50 (0 rows left)

The passed in `values` slice will be used as storage. So don't rely on the contents besides fetching the data from it.  
Therefore, you should not reference an item in the `values` slice since it will most likely be replaced sooner or later.
(depending on its size)