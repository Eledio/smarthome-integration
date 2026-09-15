# Pool Controller JSON-RPC API

The Pool Controller provides a local JSON-RPC API for reading the current device state and controlling supported functions.

The Pool Controller must be reachable from the client application over the local network. The Pool Controller can be connected via Ethernet or Wi-Fi.

The API is exposed over HTTP at the `/ubus` endpoint using JSON-RPC 2.0.

The password required for authentication is printed on the device label.

The Pool Controller uses the AWS IoT Device Shadow data model internally. The local JSON-RPC API provides direct access to the same shadow state without requiring communication with AWS IoT Core.

The device exposes its current state and configuration through a set of **shadow parameters**. The availability of individual parameters depends on the device configuration and connected peripherals. Therefore, not every parameter is necessarily available on every Pool Controller.

The shadow state contains two main sections: `reported` and `desired`.

* `reported` contains the latest values reported by the Pool Controller.
* `desired` contains requested values for writable parameters.

Both read-only and read/write parameters may be present in `reported`. Only parameters marked as read/write may be modified by setting their corresponding value in `desired`.

For read/write parameters, the value in `reported` represents the current state of the Pool Controller, while the value in `desired` represents the requested state.

The most important shadow parameters are listed below.

## Security note

The local API uses HTTP and is intended for use only on trusted local networks. Do not expose the `/ubus` endpoint directly to the Internet.

## Create a ubus RPC session

Before calling the Pool Controller API, a ubus RPC session must be created using the device credentials.

```http
POST http://your-pool-controller-IP/ubus
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 40,
  "method": "call",
  "params": [
    "00000000000000000000000000000000",
    "session",
    "login",
    {
      "username": "eledio",
      "password": "password"
    }
  ]
}
```

Example response:

```json
{
  "jsonrpc": "2.0",
  "id": 40,
  "result": [
    0,
    {
      "ubus_rpc_session": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    }
  ]
}
```

The second element of the `result` array contains the login response. The `ubus_rpc_session` value is the session token that must be used in subsequent ubus RPC calls.

The first element of the `result` array is the ubus status code. A value of `0` indicates a successful call.

The session token may expire. If a call fails because the session is no longer valid, authenticate again and obtain a new session token.

## Shadow parameters

| # | Shadow key | Description | Unit | Typical operating range / values | Access |
| --- | --- | --- | --- | --- | --- |
| 1 | `poolTemp` | Current pool water temperature | °C | 10–40 | read-only |
| 2 | `pH` | Current pool water pH level | pH | 7–8 | read-only |
| 3 | `orp` | Oxidation-reduction potential (ORP) of the pool water | mV | 100–800 | read-only |
| 4 | `freeCl` | Free chlorine concentration | ppm | 0.0–2.0 | read-only |
| 5 | `heaterStatePercent` | Current heater/cooler output level | % | 0–100 | read-only |
| 6 | `heaterMode` | Current heater operating mode | enum | `HEATING`, `COOLING` | read-only |
| 7 | `filtrationState` | Filtration pump state | boolean | `true`, `false` | **read/write** |
| 8 | `poolWaterLevel` | Current pool water level | m | 0–1.5 | read-only |
| 9 | `dosingPumpARemainingAgentVolume` | Remaining volume of pH-reducing agent | L | 0–60 | **read/write** |
| 10 | `dosingPumpBRemainingAgentVolume` | Remaining volume of chlorinating agent (NaClO) | L | 0–60 | **read/write** |

The ranges shown above represent typical operating ranges rather than strict API limits. Values outside these ranges may still be reported by the Pool Controller.

The actual availability and valid values of individual parameters may depend on the Pool Controller configuration and installed peripherals.

## Get the Thing Shadow

The current Thing Shadow can be retrieved using the `GetThingShadow` method.

```http
POST http://your-pool-controller-IP/ubus
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "call",
  "params": [
    "ubus_rpc_session",
    "eledio-app",
    "GetThingShadow",
    {}
  ]
}
```

Replace `ubus_rpc_session` with the session token obtained during login.

Example response:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": [
    0,
    {
      "state": {
        "desired": {
          "filtrationState": true
        },
        "reported": {
          "poolTemp": 27.4,
          "filtrationState": true
        }
      },
      "version": 27,
      "timestamp": 1758037606
    }
  ]
}
```

The first element of the `result` array is the ubus status code. The second element contains the result returned by the `GetThingShadow` method.

Both read-only and read/write parameters may be available through `reported`. Writable parameters can be changed by updating their corresponding value in `desired`.

The exact set of returned parameters depends on the device configuration and connected peripherals.

## Update the Thing Shadow

Writable shadow parameters can be changed using the `UpdateThingShadow` method.

The following example requests the filtration pump to be switched off:

```http
POST http://your-pool-controller-IP/ubus
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "call",
  "params": [
    "ubus_rpc_session",
    "eledio-app",
    "UpdateThingShadow",
    {
      "payload": {
        "state": {
          "desired": {
            "filtrationState": false
          }
        }
      }
    }
  ]
}
```

Example response:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": [
    0,
    {
      "state": {
        "desired": {
          "filtrationState": false
        }
      },
      "version": 28,
      "timestamp": 1758038453
    }
  ]
}
```

The response confirms that the requested value has been written to the `desired` state. It does not necessarily confirm that the requested change has already been applied by the Pool Controller.

The Pool Controller then processes the requested change and updates the corresponding `reported` state if and when the requested state is applied.

For example, after setting:

```json
{
  "filtrationState": false
}
```

in `desired`, a subsequent `GetThingShadow` call can be used to verify that `reported.filtrationState` has changed to `false`.

Only parameters marked as **read/write** should be modified through `UpdateThingShadow`. Read-only parameters represent values reported by the Pool Controller and should not be written to the `desired` state.
