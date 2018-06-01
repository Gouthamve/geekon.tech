+++
title = "TSDB: Bringing time-series into your app."
draft = false
date = "2017-05-29T20:31:02+05:30"

slug = "tsdb-embeddable-timeseries-database"

socialsharing = true
totop = true

author = "Goutham"
authortwitter = "https://twitter.com/putadent"

categories = ["prometheus", "databases", "golang"]

+++

# Introduction 
So I just finished my first two weeks as an intern with CoreOS (It’s been amazing!!). In the last two weeks, I successfully implemented deletions in tsdb, Prometheus' new time-series database. While I wanted to write about what I did, I realised you needed to know what it is I am dealing with first.

# Prologue
I am working on a [tsdb](https://github.com/prometheus/tsdb). This blog post: https://fabxc.org/blog/2017-04-10-writing-a-tsdb/ will give you a fair overview of the database. In fact, that blog is a prerequisite for understanding this one.

It has a lot of things in it, but at the end, it says:

> The code for the storage itself can be found in a [separate project](https://github.com/prometheus/tsdb). It's surprisingly agnostic to Prometheus itself and could be widely useful for a wider range of applications looking for an efficient local storage time series database.

Yep, tsdb is a _Go package_, and an _embeddable_ time-series database and you can use it in your applications. This blog post talks about the interface it provides.

# TSDB
Before we dive into the interface, let us understand some terminology:

**Labels**: Labels are key-value pairs and a set of a labels will uniquely identify a time-series (see below). 
For example: 
```path="/foo/bar/"``` is a label and 
```
{name="codelab_api_requests_total", method="POST", path="/foo/bar/"}
``` 
identifies a time-series.

**A Series**: A series (or a time-series) is a series of tuples, each tuple being a timestamp and a value, identified by a set of labels.
For example:
```
labels -> (t0, v0), (t1, v1), (t2, v2), (t3, v3), ....
```
Now our data model involves inserting data points into several time series and being able to query for different series and their data points.

## "Creating" the database
So we store all the data in a directory. We simply need to create an empty directory (tsdb directory) or choose an existing tsdb dir and need to call [`db.Open`](https://godoc.org/github.com/prometheus/tsdb#Open) on that directory.
```
db, err := tsdb.Open(path, nil, r, &tsdb.Options{
  WALFlushInterval:  10 * time.Second,
  MinBlockDuration:  uint64(15 * 60 * 1000), // 15mins in milliseconds
  MaxBlockDuration:  uint64(4 * 60 * 60 * 1000), // 4h
  RetentionDuration: uint64(15 * 24 * 60 * 60 * 1000), // 15d
})
```

There are several parameters here and let us first understand the structure of the database before we go to the parameters.

So the database is structured as `Blocks` of data with each block containing the data for a time range. While this is no longer the exact structure for the data directory (it changed very recently), it represents how the data is organised:
```
➜  prometheus git:(dev-2.0) tree data 
data
├── 01BH8A9V27EYAEVAV0FH92E27W
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01BH8YX0M8XAGH0KBRZ7X61MF0
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01BH9KG6CZYBXVNHDQSTPXJD9X
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
└── 01BH9KG6153Y4DE4ZMYE2FD2FD
     ├── meta.json
     └── wal
        └── 000001
```
So the blocks contain data in non-overlapping intervals of time. Now whenever a new datapoint is ingested, it is written to a Write Ahead Log (WAL) before being added to an in-memory (head) block (like `01BH9KG6153Y4DE4ZMYE2FD2FD`). We use a WAL here so that incase of crashes, we donot lose the in-memory data. Now after some time, the data in memory is flushed to disk as another (persisted) block like `01BH8YX0M8XAGH0KBRZ7X61MF0`. The smaller blocks are merged and compacted into larger blocks periodically.

Now, if we go back to the parameters, 

**WALFlushInterval** is the time we sync the WAL to disk. When we write a data point to the WAL, we don't actually write to disk for every single datapoint ingested, but instead write to the OS buffers and `fsync` them every `WALFlushInterval`. This means if the interval is `10s`, then in case of a crash, we might lose up to `10s` of data. By setting it to `0`, we always persist, but that comes at a performance cost.

**MinBlockDuration** is the duration after which we write a head block out as a persisted block. After writing out an existing head block, we just open another one.

**MaxBlockDuration** As we compact more and more blocks, we do not want a block to be too large. We specify the maximum interval a block covers.

**RetentionDuration** We only store data until a certain age and we drop all data beyond the retention time. If we find that a block is beyond the retention-time, we just nuke it.

## Inserting Data
Now that we know how to create or open a DB, let us throw in some data. Instead of adding a single data point, we can add several at once in a "transaction".

```
app := db.Appender()
_, err = app.Add(labels.FromStrings("foo", "bar"), 0, 0) // Handle the error when using it.
app.Add(labels.FromStrings("foo", "baz"), 0, 0)
app.Add(labels.FromStrings("foo", "fifi"), 0, 0)
app.Add(labels.Labels{{"a", "b"}}, 10, 0.5)
if err := app.Commit(); err != nil {
  // Handle error
}
```
So we created an `Appender` and appended and committed the new values. We can also choose to roll back via `err = app.Rollback()`.

## Querying Data

Now that we inserted the data, let’s read it back. This is where `tsdb` excels, by giving you a really powerful way to query time series.

We first need to specify the time range over which we need the data and then we use several `Matchers` to choose the series for which we want the data for. 
```
1  q := db.Querier(10, 1000)  // The data b/w t=10 to t=1000
2  defer q.Close()  // To release locks.
3 
4  seriesSet := q.Select(labels.NewEqualMatcher("a", "b"))
5  for seriesSet.Next() {
6    // Get each Series
7    s := seriesSet.At()
8    fmt.Println("Labels:", s.Labels())
9    fmt.Println("Data:")
10    it := s.Iterator()
11    for it.Next() {
12      ts, v := it.At()
13      fmt.Println("ts =",ts, "v =", v)
14    }
15    if err := it.Err(); err != nil {
16      // Handle error
17    }
18  }
19  if err := seriesSet.Err(); err != nil {
20    // Handle error
21  }
```

Now let’s see what is happening in each line: <br/>
**L1**: We are choosing the time-range over which to query the values. <br/>
**L4**: We are getting a [`SeriesSet`](https://godoc.org/github.com/prometheus/tsdb#SeriesSet) which matches the given [`Matcher(s)`](https://godoc.org/github.com/prometheus/tsdb/labels#Matcher). 
The signature of [`Select`](https://godoc.org/github.com/prometheus/tsdb#Querier) is `Select(...labels.Matcher) SeriesSet`

This is where the beauty lies. We are providing a bunch of selection parameters and we are receiving all the series that match those parameters. The best part is that `Matcher` is an interface, meaning you can implement your own `Matcher`s:
```
type Matcher interface {
    // Name returns the label name the matcher should apply to.
    Name() string
    // Matches checks whether a value fulfills the constraints.
    Matches(v string) bool
}
```
Some of the Existing `Matcher`s include `EqualMatcher`, `RegexpMatcher` and `NotMatcher`.
Illustrating some selection examples:
```
// Select any series having `{path="foo"}` as a label.
eqm := labels.NewEqualMatcher("path", "foo")
ss := q.Select(eqm)

// Select all metrics where path matches "foo.*" ({path=~"foo.*"})
rem, _ := labels.NewRegexpMatcher("path", "foo.*") // Do not ignore error in your code :P.
ss := q.Select(rem)

// Select all metrics where path does not match "foo.*"
ss := q.Select(labels.Not(rem))

// Select all metrics having `{path="foo", method="POST"}` as labels (both of them).
ss := q.Select(eqm, labels.NewEqualMatcher("method", "POST"))
```

**L5-21**: We are now going over the set of series and are extracting the series data (its labels and data points). It is to be noted that the series are in sorted order of their labels.<br/>
**L10-17**: We use the [`SeriesIterator`](https://godoc.org/github.com/prometheus/tsdb#SeriesIterator) interface to iterate over the data points (which are sorted according to time, duh!).

## Deleting Data
Now finally to my work! This was what I was working on for the past two weeks, to add an API to delete data.

To delete all data between timestamp 10 and 1000 in all series having labels `{path="foo", method="POST"}`, you simply do:
```
err := db.Delete(
  10, // mint
  1000, // maxt
  labels.NewEqualMatcher("path", "/foo"), // matchers
  labels.NewEqualMatcher("method", "POST"),
)
```
And boom! the data will be gone. 

## The Complete Picture
A full runnable example of everything covered above is presented here:
<code data-gist-id="52744e33f60ef676d8b523a3c1f21d86"></code>

# Epilogue
So this is just an introduction to the database and I plan to cover the internals and benchmarks in the coming weeks. But before you go off, I think this can be used in a LOT of places, and if you have any feedback or will use tsdb, please comment it below or say “Hi!” in the freenode IRC channel #prometheus.

Finally, I want to thank [Fabian](https://github.com/fabxc) for patiently reviewing my [huuuuge PR](https://github.com/prometheus/tsdb/pull/82) _several_ times.


<script type="text/javascript" src="https://ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/gist-embed/2.4/gist-embed.min.js"></script>
