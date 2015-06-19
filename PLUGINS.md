Telegraf is entirely plugin driven. This interface allows for operators to
pick and chose what is gathered as well as makes it easy for developers
to create new ways of generating metrics.

Plugin authorship is kept as simple as possible to promote people to develop
and submit new plugins.

## Guidelines

* A plugin must conform to the `plugins.Plugin` interface.
* Telegraf promises to run each plugin's Gather function serially. This means
developers don't have to worry about thread safety within these functions.
* Each generated metric automatically has the name of the plugin that generated
it prepended. This is to keep plugins honest.
* Plugins should call `plugins.Add` in their `init` function to register themselves.
See below for a quick example.
* To be available within Telegraf itself, plugins must add themselves to the `github.com/influxdb/telegraf/plugins/all/all.go` file.
* The `SampleConfig` function should return valid toml that describes how the plugin can be configured. This is include in `telegraf -sample-config`.
* The `Description` function should say in one line what this plugin does.

### Plugin interface

```go
type Plugin interface {
	SampleConfig() string
	Description() string
	Gather(Accumulator) error
}

type Accumulator interface {
  Add(measurement string, value interface{}, tags map[string]string)
  AddValuesWithTime(measurement string, values map[string]interface{}, tags map[string]string, timestamp time.Time)
}
```

### Accumulator

The way that a plugin emits metrics is by interacting with the Accumulator.

The `Add` function takes 3 arguments:
* **measurement**: A string description of the metric. For instance `bytes_read` or `faults`.
* **value**: A value for the metric. This accepts 5 different types of value:
  * **int**: The most common type. All int types are accepted but favor using `int64`
  Useful for counters, etc.
  * **float**: Favor `float64`, useful for gauges, percentages, etc.
  * **bool**: `true` or `false`, useful to indicate the presence of a state. `light_on`, etc.
  * **string**: Typically used to indicate a message, or some kind of freeform information.
  * **time.Time**: Useful for indicating when a state last occurred, for instance `light_on_since`.
* **tags**: This is a map of strings to strings to describe the where or who about the metric. For instance, the `net` plugin adds a tag named `"interface"` set to the name of the network interface, like `"eth0"`.

The `AddValuesWithTime` allows multiple values for a point to be passed. The values
used are the same type profile as **value** above. The **timestamp** argument
allows a point to be registered as having occurred at an arbitrary time.

Let's say you've written a plugin that emits metrics about processes on the current host.

```go

type Process struct {
  CPUTime float64
  MemoryBytes int64
  PID int
}

func Gather(acc plugins.Accumulator) error {
  for _, process := range system.Processes() {
    tags := map[string]string {
      "pid": fmt.Sprintf("%d", process.Pid),
    }

    acc.Add("cpu", process.CPUTime, tags)
    acc.Add("memoory", process.MemoryBytes, tags)
  }
}
```

### Example

```go

// simple.go

import "github.com/influxdb/telegraf/plugins"

type Simple struct {
  Ok bool
}

func (s *Simple) Description() string {
  return "a demo plugin"
}

func (s *Simple) SampleConfig() string {
  return "ok = true # indicate if everything is fine"
}

func (s *Simple) Gather(acc plugins.Accumulator) error {
  if s.Ok {
    acc.Add("state", "pretty good", nil)
  } else {
    acc.Add("state", "not great", nil)
  }

  return nil
}

func init() {
  plugins.Add("simple", func() plugins.Plugin { &Simple{} })
}
```
