# Gleam
[![Build Status](https://travis-ci.org/chrislusf/gleam.svg?branch=master)](https://travis-ci.org/chrislusf/gleam)
[![GoDoc](https://godoc.org/github.com/chrislusf/gleam/flow?status.svg)](https://godoc.org/github.com/chrislusf/gleam/flow)
[![Wiki](https://img.shields.io/badge/docs-wiki-blue.svg)](https://github.com/chrislusf/gleam/wiki)
[![Go Report Card](https://goreportcard.com/badge/github.com/chrislusf/gleam)](https://goreportcard.com/report/github.com/chrislusf/gleam)
[![codecov](https://codecov.io/gh/chrislusf/gleam/branch/master/graph/badge.svg)](https://codecov.io/gh/chrislusf/gleam)

Gleam is a high performance and efficient distributed execution system, and also
simple, generic, flexible and easy to customize.

Gleam is built in Go, and the user defined computation can be written in Go, 
Unix pipe tools, or any streaming programs.

### High Performance

* Pure Go mappers and reducers have high performance and concurrency.
* Data flows through memory, optionally to disk.
* Multiple map reduce steps are merged together for better performance.


### Memory Efficient

* Gleam does not have the common GC problem that plagued other languages. Each executor runs in a separated OS process. The memory is managed by the OS. One machine can host many more executors.
* Gleam master and agent servers are memory efficient, consuming about 10 MB memory.
* Gleam tries to automatically adjust the required memory size based on data size hints, avoiding the try-and-error manual memory tuning effort.

### Flexible
* The Gleam flow can run standalone or distributed.
* Adjustable in memory mode or OnDisk mode.

### Easy to Customize
* The Go code is much simpler to read than Scala, Java, C++.

# One Flow, Multiple ways to execute
Gleam code defines the flow, specifying each dataset(vertex) and computation step(edge), and build up a directed
acyclic graph(DAG). There are multiple ways to execute the DAG.

The default way is to run locally. This works in most cases.

Here we mostly talk about the distributed mode.

## Distributed Mode
The distributed mode has several names to explain: Master, Agent, Executor, Driver.

### Gleam Driver

* Driver is the program users write, it defines the flow, and talks to Master, Agents, and Executors.

### Gleam Master

* The Master is one single server that collects resource information from Agents.
* It stores transient resource information and can be restarted.
* When the Driver program starts, it asks the Master for available Executors on Agents.

### Gleam Agent

* Agents runs on any machine that can run computations.
* Agents periodically send resource usage updates to Master.
* When the Driver program has executors assigned, it talks to the Agents to start Executors.
* Agents also manage datasets generated by each Executors.

### Gleam Executor
* Executors are started by Agents. They will read inputs from external or previous datasets, process them, and output to a new dataset.

### Dataset

* The datasets are managed by Agents. By default, the data run only through memory and network, not touching slow disk.
* Optionally the data can be persist to disk.

By leaving it in memory, the flow can have back pressure, and can support stream computation naturally.

# Documentation
* [Gleam Wiki](https://github.com/chrislusf/gleam/wiki)
* [Installation](https://github.com/chrislusf/gleam/wiki/Installation)
* [Gleam Flow API GoDoc](https://godoc.org/github.com/chrislusf/gleam/flow)
* [gleam-dev on Slack](https://gleam-dev.slack.com)

# Standalone Example

## Word Count

#### Word Count

Basically, you need to register the Go functions first.
It will return a mapper or reducer function id, which we can pass it to the flow.

```go
package main

import (
	"flag"
	"strings"

	"github.com/chrislusf/gleam/distributed"
	"github.com/chrislusf/gleam/flow"
	"github.com/chrislusf/gleam/gio"
	"github.com/chrislusf/gleam/plugins/file"
)

var (
	isDistributed   = flag.Bool("distributed", false, "run in distributed or not")
	Tokenize  = gio.RegisterMapper(tokenize)
	AppendOne = gio.RegisterMapper(appendOne)
	Sum = gio.RegisterReducer(sum)
)

func main() {

	gio.Init()   // If the command line invokes the mapper or reducer, execute it and exit.
	flag.Parse() // optional, since gio.Init() will call this also.

	f := flow.New("top5 words in passwd").
		Read(file.Txt("/etc/passwd", 2)).  // read a txt file and partitioned to 2 shards
		Map("tokenize", Tokenize).    // invoke the registered "tokenize" mapper function.
		Map("appendOne", AppendOne).  // invoke the registered "appendOne" mapper function.
		ReduceBy("sum", Sum).         // invoke the registered "sum" reducer function.
		Sort("sortBySum", flow.OrderBy(2, true)).
		Top("top5", 5, flow.OrderBy(2, false)).
		Printlnf("%s\t%d")

	if *isDistributed {
		f.Run(distributed.Option())
	} else {
		f.Run()
	}

}

func tokenize(row []interface{}) error {
	line := gio.ToString(row[0])
	for _, s := range strings.FieldsFunc(line, func(r rune) bool {
		return !('A' <= r && r <= 'Z' || 'a' <= r && r <= 'z' || '0' <= r && r <= '9')
	}) {
		gio.Emit(s)
	}
	return nil
}

func appendOne(row []interface{}) error {
	row = append(row, 1)
	gio.Emit(row...)
	return nil
}

func sum(x, y interface{}) (interface{}, error) {
	return gio.ToInt64(x) + gio.ToInt64(y), nil
}

```

Now you can execute the binary directly or with "-distributed" option to run in distributed mode.
The distributed mode would need a simple setup described later.

A bit more blown up example is here, using the predefined mapper or reducer:
https://github.com/chrislusf/gleam/blob/master/examples/word_count_in_go/word_count_in_go.go


#### Word Count by Unix Pipe Tools
Here is another way to do the similar by unix pipe tools.

Unix Pipes are easy for sequential pipes, but limited to fan out, and even more limited to fan in.

With Gleam, fan-in and fan-out parallel pipes become very easy.

```go
package main

import (
	"fmt"

	"github.com/chrislusf/gleam/flow"
	"github.com/chrislusf/gleam/gio"
	"github.com/chrislusf/gleam/gio/mapper"
	"github.com/chrislusf/gleam/plugins/file"
	"github.com/chrislusf/gleam/util"
)

func main() {

	gio.Init()

	flow.New("word count by unix pipes").
		Read(file.Txt("/etc/passwd", 2)).
		Map("tokenize", mapper.Tokenize).
		Pipe("lowercase", "tr 'A-Z' 'a-z'").
		Pipe("sort", "sort").
		Pipe("uniq", "uniq -c").
		OutputRow(func(row *util.Row) error {

			fmt.Printf("%s\n", gio.ToString(row.K[0]))

			return nil
		}).Run()

}
```

This example used OutputRow() to process the output row directly.

## Join two CSV files.

Assume there are file "a.csv" has fields "a1, a2, a3, a4, a5" 
and file "b.csv" has fields "b1, b2, b3". 
We want to join the rows where a1 = b2. 
And the output format should be "a1, a4, b3".

```go
package main

import (
	. "github.com/chrislusf/gleam/flow"
	"github.com/chrislusf/gleam/gio"
	"github.com/chrislusf/gleam/plugins/file"
)

func main() {

	gio.Init()

	f := New("join a.csv and b.csv by a1=b2")
	a := f.Read(file.Csv("a.csv", 1)).Select("select", Field(1,4)) // a1, a4
	b := f.Read(file.Csv("b.csv", 1)).Select("select", Field(2,3)) // b2, b3

	a.Join("joinByKey", b).Printlnf("%s,%s,%s").Run()  // a1, a4, b3

}

```

# Distributed Computing
## Setup Gleam Cluster Locally
Start a gleam master and several gleam agents
```go
// start "gleam master" on a server
> go get github.com/chrislusf/gleam/distributed/gleam
> gleam master --address=":45326"

// start up "gleam agent" on some different servers or ports
> gleam agent --dir=2 --port 45327 --host=127.0.0.1
> gleam agent --dir=3 --port 45328 --host=127.0.0.1
```

## Setup Gleam Cluster on Kubernetes
Start a gleam master and several gleam agents
```bash
kubectl apply -f k8s/
```

## Change Execution Mode.

After the flow is defined, the Run() function can be executed in local mode or distributed mode.

```go
  f := flow.New("")
  ...
  // 1. local mode
  f.Run()

  // 2. distributed mode
  import "github.com/chrislusf/gleam/distributed"
  f.Run(distributed.Option())
  f.Run(distributed.Option().SetMaster("master_ip:45326"))

```

# Important Features

* Fault tolerant [OnDisk()](https://godoc.org/github.com/chrislusf/gleam/flow#Dataset.OnDisk).
* Read data from Local, HDFS, or S3.
* Data Sources
  * [Cassandra](https://github.com/chrislusf/gleam/tree/master/plugins/cassandra), with [example](https://github.com/chrislusf/gleam/tree/master/examples/cassandra_reader)
  * [Kafka](https://github.com/chrislusf/gleam/tree/master/plugins/kafka)
  * [ORC files](https://github.com/chrislusf/gleam/tree/master/plugins/file/orc)
  * [CSV files](https://github.com/chrislusf/gleam/tree/master/plugins/file/csv)
  * [TSV files](https://github.com/chrislusf/gleam/tree/master/plugins/file/tsv)
  * [TXT files](https://github.com/chrislusf/gleam/tree/master/plugins/file/txt)
  * Raw Socket

# Status
Gleam is just beginning. Here are a few todo items. Welcome any help!
* [Add new plugin to read external data](https://github.com/chrislusf/gleam/wiki/Add-New-Source).
* Add windowing functions similar to Apache Beam/Flink. (in progress)
* Add schema support for each dataset.
* Support using SQL as a flow step, similar to LINQ.
* Add dataset metadata for better caching of often re-calculated data.

Especially Need Help Now:
* Go implementation to read Parquet files.

Please start to use it and give feedback. Help is needed. Anything is welcome. Small things count: fix documentation, adding a logo, adding docker image, blog about it, share it, etc.

[![](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=EEECLJ8QGTTPC)

## License

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
