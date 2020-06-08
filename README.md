# envconfig

An extension of [http://godoc.org/github.com/kelseyhightower/envconfig](http://godoc.org/github.com/kelseyhightower/envconfig)

```Go
import "github.com/tanqhnguyen/envconfig"
```

## Documentation

See [godoc](http://godoc.org/github.com/tanqhnguyen/envconfig)

## Usage

Set some environment variables:

```Bash
export MYAPP_DEBUG=false
export MYAPP_PORT=8080
export MYAPP_USER=Kelsey
export MYAPP_RATE="0.5"
export MYAPP_TIMEOUT="3m"
export MYAPP_USERS="rob,ken,robert"
export MYAPP_COLORCODES="red:1,green:2,blue:3"
```

Write some code:

```Go
package main

import (
    "fmt"
    "log"
    "time"

    "github.com/kelseyhightower/envconfig"
)

type Specification struct {
    Debug       bool
    Port        int
    User        string
    Users       []string
    Rate        float32
    Timeout     time.Duration
    ColorCodes  map[string]int
}

func main() {
    var s Specification
    err := envconfig.Process("myapp", &s)
    if err != nil {
        log.Fatal(err.Error())
    }
    format := "Debug: %v\nPort: %d\nUser: %s\nRate: %f\nTimeout: %s\n"
    _, err = fmt.Printf(format, s.Debug, s.Port, s.User, s.Rate, s.Timeout)
    if err != nil {
        log.Fatal(err.Error())
    }

    fmt.Println("Users:")
    for _, u := range s.Users {
        fmt.Printf("  %s\n", u)
    }

    fmt.Println("Color codes:")
    for k, v := range s.ColorCodes {
        fmt.Printf("  %s: %d\n", k, v)
    }
}
```

Results:

```Bash
Debug: false
Port: 8080
User: Kelsey
Rate: 0.500000
Timeout: 3m0s
Users:
  rob
  ken
  robert
Color codes:
  red: 1
  green: 2
  blue: 3
```

## Struct Tag Support

Envconfig supports the use of struct tags to specify alternate, default, content from file and required
environment variables.

For example, consider the following struct:

```Go
type Specification struct {
    ManualOverride1 string `envconfig:"manual_override_1"`
    DefaultVar      string `default:"foobar"`
    RequiredVar     string `required:"true"`
    IgnoredVar      string `ignored:"true"`
    AutoSplitVar    string `split_words:"true"`
    RequiredAndAutoSplitVar    string `required:"true" split_words:"true"`
    Secret          string `file_content:"true"`
}
```

Envconfig has automatic support for CamelCased struct elements when the
`split_words:"true"` tag is supplied. Without this tag, `AutoSplitVar` above
would look for an environment variable called `MYAPP_AUTOSPLITVAR`. With the
setting applied it will look for `MYAPP_AUTO_SPLIT_VAR`. Note that numbers
will get globbed into the previous word. If the setting does not do the
right thing, you may use a manual override.

Envconfig will process value for `ManualOverride1` by populating it with the
value for `MYAPP_MANUAL_OVERRIDE_1`. Without this struct tag, it would have
instead looked up `MYAPP_MANUALOVERRIDE1`. With the `split_words:"true"` tag
it would have looked up `MYAPP_MANUAL_OVERRIDE1`.

```Bash
export MYAPP_MANUAL_OVERRIDE_1="this will be the value"

# export MYAPP_MANUALOVERRIDE1="and this will not"
```

If envconfig can't find an environment variable value for `MYAPP_DEFAULTVAR`,
it will populate it with "foobar" as a default value.

If envconfig can't find an environment variable value for `MYAPP_REQUIREDVAR`,
it will return an error when asked to process the struct. If
`MYAPP_REQUIREDVAR` is present but empty, envconfig will not return an error.

If envconfig can't find an environment variable in the form `PREFIX_MYVAR`, and there
is a struct tag defined, it will try to populate your variable with an environment
variable that directly matches the envconfig tag in your struct definition:

```shell
export SERVICE_HOST=127.0.0.1
export MYAPP_DEBUG=true
```

```Go
type Specification struct {
    ServiceHost string `envconfig:"SERVICE_HOST"`
    Debug       bool
}
```

Envconfig won't process a field with the "ignored" tag set to "true", even if a corresponding
environment variable is set.

The tag `file_content` can be used to read from a file, this is useful when we need to use [docker secret](https://docs.docker.com/engine/swarm/secrets/). By default, it appends `_FILE` to the env variable name to look for the file path. For example, with this struct, envconfig will look for `MYAPP_SECRET_FILE`, change `file_content` to something if needed (for instance, `file_content:"__FILE"`, this will look for `MYAPP_SECRET__FILE`). Content read from a file will have higher priority than value stored in `MYAPP_SECRET`. If the file is not found / unreadable for whatever reason, it will fail gracefully and fallback to `MYAPP_SECRET`.

## Supported Struct Field Types

envconfig supports these struct field types:

- string
- int8, int16, int32, int64
- bool
- float32, float64
- slices of any supported type
- maps (keys and values of any supported type)
- [encoding.TextUnmarshaler](https://golang.org/pkg/encoding/#TextUnmarshaler)
- [encoding.BinaryUnmarshaler](https://golang.org/pkg/encoding/#BinaryUnmarshaler)
- [time.Duration](https://golang.org/pkg/time/#Duration)

Embedded structs using these fields are also supported.

## Custom Decoders

Any field whose type (or pointer-to-type) implements `envconfig.Decoder` can
control its own deserialization:

```Bash
export DNS_SERVER=8.8.8.8
```

```Go
type IPDecoder net.IP

func (ipd *IPDecoder) Decode(value string) error {
    *ipd = IPDecoder(net.ParseIP(value))
    return nil
}

type DNSConfig struct {
    Address IPDecoder `envconfig:"DNS_SERVER"`
}
```

Also, envconfig will use a `Set(string) error` method like from the
[flag.Value](https://godoc.org/flag#Value) interface if implemented.
