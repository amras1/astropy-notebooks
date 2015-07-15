# Index profiling

## Memory profiling

For a 100,000 line table with one column of randomly generated numbers, the required memory to create an index is about 4 times the table memory for FastRBT:
```
Line #    Mem usage    Increment   Line Contents
================================================
     9   38.102 MiB    0.000 MiB   @profile
    10                             def create_index():
    11   44.273 MiB    6.172 MiB       t = Table([[randint(0, N*10) for i in range(N)]])
    12   68.629 MiB   24.355 MiB       t.add_index('col0', impl=FastRBT)
    
```
And about 2/3 of the table memory for SortedArray:
```
Line #    Mem usage    Increment   Line Contents
================================================
     9   38.105 MiB    0.000 MiB   @profile
    10                             def create_index():
    11   44.281 MiB    6.176 MiB       t = Table([[randint(0, N*10) for i in range(N)]])
    12   48.250 MiB    3.969 MiB       t.add_index('col0', impl=SortedArray)

```

## Time profiling

Here are the test functions I use (again, I'm using a 100,000 line table):
```
N = 100000

class IndexProfiling:
    def __init__(self, impl):
        # initialize N rows with random (mostly unique) elements
        self.t = Table([[randint(0, N * 10) for i in range(N)]])
        self.impl = impl

    def time_init(self):
        if self.impl is not None:
            self.t.add_index('col0', impl=self.impl)

    def time_group(self):
        self.t.group_by('col0')

    def time_where(self):
        self.t.where('[col0] = {0}', N / 2)

    def time_query(self):
        self.t.query({'col0': N / 2})

    def time_query_range(self):
        # from N/4 to 3N/4, inclusive
        self.t.query({'col0': ((N / 4, 3 * N / 4), (True, True))})

    def time_add_row(self):
        self.t.add_row((randint(0, N * 10),))

    def time_modify(self):
        self.t['col0'][0] = randint(0, N * 10)
```

And the results (in seconds):
```
FastRBT
**********
time_init: 1.2263660431
time_group: 0.2325990200
time_where: 0.0003449917
time_query: 0.0000329018
time_query_range: 0.0643420219
time_add_row: 2.7397549152
time_modify: 0.0001499653

SortedArray
**********
time_init: 0.0355048180
time_group: 0.0041830540
time_where: 0.0000801086
time_query: 0.0000169277
time_query_range: 0.0217781067
time_add_row: 0.0200960636
time_modify: 0.0016808510

None
**********
time_init: 0.0000019073
time_group: 0.0865180492
time_where: 0.0002820492
time_query: 0.0001931190
time_query_range: 0.2128648758
time_add_row: 0.0006089211
time_modify: 0.0000159740
```

Note that the time required to add a row is unreasonably large for the indexed versions, which seems to be the result of deep copying of indices when a new Column object is instantiated. In FastRBT, initialization is slow because there remains a pure-Python loop to create the dict required for the bintrees library, and add_row is evidently ridiculously slow because of an unwanted recurring loop in which the index is deep copied.
