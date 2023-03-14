## bmclib v2 - board management controller library

[![Status](https://github.com/bmc-toolbox/bmclib/actions/workflows/ci.yaml/badge.svg)](https://github.com/bmc-toolbox/bmclib/actions)
[![Go Report Card](https://goreportcard.com/badge/github.com/bmc-toolbox/bmclib)](https://goreportcard.com/report/github.com/bmc-toolbox/bmclib/v2)
[![GoDoc](https://godoc.org/github.com/bmc-toolbox/bmclib/v2?status.svg)](https://godoc.org/github.com/bmc-toolbox/bmclib/v2)

bmclib v2 is a library to abstract interacting with baseboard management controllers.

### Supported BMC interfaces.

 - [Redfish](https://github.com/bmc-toolbox/bmclib/tree/main/providers/redfish)
 - [IPMItool](https://github.com/bmc-toolbox/bmclib/tree/main/providers/ipmitool)
 - [Intel AMT](https://github.com/bmc-toolbox/bmclib/tree/main/providers/intelamt)
 - [Asrockrack](https://github.com/bmc-toolbox/bmclib/tree/main/providers/asrockrack)

### Installation

```bash
go get github.com/bmc-toolbox/bmclib/v2
```

### Import 

```go
import (
  bmclib "github.com/bmc-toolbox/bmclib/v2"
)
```

### Usage

The snippet below connects to a BMC and retrieves the device hardware, firmware inventory.

```go
import (
  bmclib "github.com/bmc-toolbox/bmclib/v2"
)

    // setup logger
    l := logrus.New()
    l.Level = logrus.DebugLevel
    logger := logrusr.New(l)

    clientOpts := []bmclib.Option{bmclib.WithLogger(logger)}

    // init client
    client := bmclib.NewClient(*host, "", "admin", "hunter2", clientOpts...)

    // open BMC session
    err := client.Open(ctx)
    if err != nil {
        log.Fatal(err, "bmc login failed")
    }

    defer client.Close(ctx)

    // retrieve inventory data
    inventory, err := client.Inventory(ctx)
    if err != nil {
        l.Error(err)
    }

    b, err := json.MarshalIndent(inventory, "", "  ")
    if err != nil {
        l.Error(err)
    }

    fmt.Println(string(b))
```

More sample code can be found in [examples](./examples/)

### BMC connections

bmclib performs queries on BMCs using [multiple `drivers`](https://github.com/bmc-toolbox/bmclib/blob/main/bmc/connection.go#L30),
these `drivers` are the various services exposed by a BMC - `redfish` `IPMI` `SSH` and `vendor API` which is basically a custom vendor API endpoint.

The bmclib client determines which driver to use for an action like `Power cycle` or `Create user`
based on its availability or through a compatibility test (when enabled).

When querying multiple BMCs through bmclib its often useful to to limit the BMCs and
drivers that bmclib will attempt to use to connect, the options to limit or filter
out BMCs are described below,

Query just using the `redfish` endpoint.
```
cl := bmclib.NewClient("192.168.1.1", "", "admin", "hunter2")
cl.Registry.Drivers = cl.Registry.Using("redfish")
```

Query using the `redfish` endpoint and fall back to `IPMI`
```
client := bmclib.NewClient("192.168.1.1", "", "admin", "hunter2")

// overwrite registered drivers by appending Redfish, IPMI drivers in order
drivers := append(registrar.Drivers{}, bmcClient.Registry.Using("redfish")...)
drivers = append(drivers, bmcClient.Registry.Using("ipmi")...)
client.Registry.Drivers = driver
```

Filter drivers to query based on compatibility, this will attempt to check if the driver is
[compatible](https://github.com/bmc-toolbox/bmclib/blob/main/providers/redfish/redfish.go#L70)
ideally, this method should be invoked when the client is ready to perform a BMC action.
```
client := bmclib.NewClient("192.168.1.1", "", "admin", "hunter2")
client.Registry.Drivers = cl.Registry.FilterForCompatible(ctx)
```

Ignore the Redfish endpoint completely on BMCs running a specific Redfish version.

Note: this version should match the one returned through `curl -k  "https://<BMC IP>/redfish/v1" | jq .RedfishVersion`
```
opt := bmclib.WithRedfishVersionsNotCompatible([]string{"1.5.0"})

client := bmclib.NewClient("192.168.1.1", "", "admin", "hunter2", opt...)
cl.Registry.Drivers = cl.Registry.FilterForCompatible(ctx)
```

### Timeouts

bmclib can be configured to apply timeouts to BMC interactions. The following options are available.

**Total max timeout only** - The total time bmclib will wait for all BMC interactions to complete. This is specified using a single `context.WithTimeout` or `context.WithDeadline` that is passed to all method call. With this option, the per provider; per interaction timeout is calculated by the total max timeout divided by the number of providers (currently there are 4 providers).

```Go
cl := bmclib.NewClient(host, port, user, pass, bmclib.WithLogger(log))

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err = cl.Open(ctx); err != nil {
  return(err)
}
defer cl.Close(ctx)

state, err := cl.GetPowerState(ctx)
```

**Total max timeout and a per provider; per interaction timeout** - The total time bmclib will wait for all BMC interactions to complete. This is specified using a single `context.WithTimeout` or `context.WithDeadline` that is passed to all method call. This is honored above all timeouts. The per provider; per interaction timeout is specified using `bmclib.WithPerProviderTimeout` in the Client constructor.

```Go
cl := bmclib.NewClient(host, port, user, pass, bmclib.WithLogger(log), bmclib.WithPerProviderTimeout(15*time.Second))

ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
defer cancel()

if err = cl.Open(ctx); err != nil {
  return(err)
}
defer cl.Close(ctx)

state, err := cl.GetPowerState(ctx)
```

**Per provider; per interaction timeout. No total max timeout** - The time bmclib will wait for a specific provider to complete. This is specified using `bmclib.WithPerProviderTimeout` in the Client constructor.

```Go
cl := bmclib.NewClient(host, port, user, pass, bmclib.WithLogger(log), bmclib.WithPerProviderTimeout(15*time.Second))

ctx := context.Background()

if err = cl.Open(ctx); err != nil {
  return(err)
}
defer cl.Close(ctx)

state, err := cl.GetPowerState(ctx)
```

**Default timeout** - If no timeout is specified with a context or with `bmclib.WithPerProviderTimeout` the default is used. 30 seconds per provider; per interaction.

```Go
cl := bmclib.NewClient(host, port, user, pass, bmclib.WithLogger(log))

ctx := context.Background()

if err = cl.Open(ctx); err != nil {
  return(err)
}
defer cl.Close(ctx)

state, err := cl.GetPowerState(ctx)
```

### bmclib versions

The current bmclib version is `v2` and is being developed on the `main` branch.

The previous bmclib version is in maintenance mode and can be found here [v1](https://github.com/bmc-toolbox/bmclib/v1).

### Acknowledgments

bmclib v2 interfaces with Redfish on BMCs through the Gofish library https://github.com/stmcginnis/gofish

bmclib was originally developed for [Booking.com](http://www.booking.com). With approval from [Booking.com](http://www.booking.com), 
the code and specification were generalized and published as Open Source on github, for which the authors would like to express their gratitude.

### Authors
- [Joel Rebello](https://github.com/joelrebel) 
- [Jacob Weinstock](https://github.com/jacobweinstock)

