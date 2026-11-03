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

The first element of the `result` array is the ubus status code. A value of `0` indicates a successful call. Any other value indicates an error. A list of error codes is not available.

### Session timeout

The session token expires when it is not used for the configured time. The default timeout is 300 seconds (5 minutes). A different timeout can be requested by adding the optional `timeout` parameter (in seconds) to the login call:

```json
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
      "password": "password",
      "timeout": 3600
    }
  ]
}
```

The example above creates a session that is valid for 3600 seconds (1 hour).

Every API request made with the session resets the timeout countdown. For example, with a timeout of 5 minutes and an API request every 3 minutes, the session never expires.

The session is stored only in the memory of the Pool Controller and does not survive a restart of the device. A session therefore cannot be made permanent: after a restart, a new login is always required.

If a call fails because the session has expired or is no longer valid, authenticate again and obtain a new session token.

## Request rate

There is no rate limit on API requests. The values in the shadow are refreshed once per second, so polling more often than every second brings no new data.

Sending API requests more often than every 5 seconds is not recommended, because it may overload the Pool Controller.

## Shadow parameters

| # | Shadow key | Description | Data type | Unit | Typical operating range / values | Access |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `poolTemp` | Current pool water temperature | number | °C | 10–40 | read-only |
| 2 | `targetTemp` | Target pool water temperature | number | °C | 0–40 (step 0.5) | **read/write** |
| 3 | `pH` | Current pool water pH level | number | pH | 7–8 | read-only |
| 4 | `orp` | Oxidation-reduction potential (ORP) of the pool water | number | mV | 100–800 | read-only |
| 5 | `freeCl` | Free chlorine concentration | number | ppm | 0.0–2.0 | read-only |
| 6 | `heaterStatePercent` | Current heater/cooler output level | number | % | 0–100 | read-only |
| 7 | `heaterMode` | Current heater operating mode (the dashboard uses it only in icon/label rules, not as a tile value) | enum | – | `HEATING`, `COOLING`, `STANDBY` | read-only |
| 8 | `filtrationState` | Filtration pump state | boolean | – | `true`, `false` | **read/write** |
| 9 | `poolWaterLevel` | Current pool water level | number | m | 0–1.5 | read-only |
| 10 | `dosingPumpARemainingAgentVolume` | Remaining volume of agent in dosing pump A (e.g. pH-reducing agent) | number | L | 0–60 | **read/write** |
| 11 | `dosingPumpBRemainingAgentVolume` | Remaining volume of agent in dosing pump B (e.g. chlorinating agent) | number | L | 0–60 | **read/write** |
| 12 | `dosingPumpCRemainingAgentVolume` | Remaining volume of agent in dosing pump C | number | L | 0–60 | **read/write** |
| 13 | `dosingPumpDRemainingAgentVolume` | Remaining volume of agent in dosing pump D | number | L | 0–60 | **read/write** |
| 14 | `ambientTemperature` | Ambient air temperature (shown only with the `displayAmbientTemperature` eligibility) | number | °C | – | read-only |
| 15 | `temperatureIntoPool` | Return water temperature (water flowing into the pool) | number | °C | – | read-only |
| 16 | `outsideTemp` | Outside air temperature | number | °C | – | read-only |
| 17 | `targetpHLevel` | Target pH level | number | pH | 7–8 (step 0.05) | **read/write** |
| 18 | `targetORPLevel` | Target ORP level | number | mV | 100–800 (step 10) | **read/write** |
| 19 | `targetFreeClLevel` | Target free chlorine concentration | number | ppm | 0–2 (step 0.05) | **read/write** |
| 20 | `dissolvedOxygen` | Dissolved oxygen (unit is not defined in the template) | number | – | – | read-only |
| 21 | `tankWaterLevel` | Current water level in the balancing tank (the UI shows it in cm) | number | m | – | read-only |
| 22 | `salinity` | Current water salinity | number | % | 0.1–1.2 | read-only |
| 23 | `chlorinatorOutputClPercent` | Current chlorinator output level | number | % | 0–100 | read-only |
| 24 | `targetChlorinatorOutputClPercent` | Target chlorinator output level | number | % | 0–100 (step 10) | **read/write** |
| 25 | `chlorinationMethod` | Chlorination method | enum | – | `NACLO`, `CHLORINATOR`, `NACLO_AND_CHLORINATOR` | **read/write** |
| 26 | `targetHeaterMode` | Requested heater operating mode | enum | – | `HEATING`, `DISABLED`, `COOLING` *(only in some configurations)*, `AUTO` *(only in some configurations)* | **read/write** |
| 27 | `startCoolingTemp` | Water temperature at which cooling starts | number | °C | 20–36 (step 0.5) | **read/write** |
| 28 | `heaterMaxPowerLevel` | Maximum heater power level | enum | – | `SILENT`, `NORMAL`, `BOOST` *(only in some configurations)* | **read/write** |
| 29 | `waterRefill` | Automatic water refill | enum | – | `DISABLED`, `ENABLED` | **read/write** |
| 30 | `aux<N>State` | State of auxiliary output N. The set of outputs depends on the switchboard type (`aux0State` … `aux7State` on CPC2 and others, `aux1State` … `aux13State` on IPC1 and others). What the output controls is set by the `aux<N>Output` parameter. Not every output can be switched directly, see ¹. | boolean | – | `true`, `false` | read-only / **read/write** ¹ |
| 31 | `lightsColorChange` | Trigger for changing the RGB light colour | toggle (trigger) | – | – | write-only |
| 32 | `protectionRCDState` | RCD protection active | boolean | – | `true`, `false` | read-only |

¹ `aux<N>State` is writable (`$desired.aux<N>State`) only for outputs of the type `lighting`, `lightingRgb`, `attraction`, `auxiliaryCirculationPump`, `bubbleBath`, `fountain`, `jetStream` and `massage`. For the other output types the state is only reported (if necessary) and the equipment is controlled by other means:
- `mainPump` – controlled by `filtrationState` and further device logic,
- `heater`, `heaterContact` – controlled by `targetTemp` / `targetHeaterMode` and the heating regulation,

The function of each auxiliary output (for example light, jet stream or fountain) is part of the configuration of the specific installation and differs between devices. To control a particular function, the client must therefore know which output it is connected to. The function cannot be derived from the number of the output.

The ranges shown above represent typical operating ranges rather than strict API limits. Values outside these ranges may still be reported by the Pool Controller.

The actual availability and valid values of individual parameters may depend on the Pool Controller configuration and installed peripherals.

## Common functions

| Function | Shadow keys |
| --- | --- |
| Pool light | `aux<N>State` of the output configured as `lighting` or `lightingRgb` (the output number depends on the installation); the colour of an RGB light is changed with `lightsColorChange` |
| Heating on/off | `targetHeaterMode` (`HEATING` = on, `DISABLED` = off); target temperature `targetTemp` |
| Switched 230 V outputs (sockets) | The sockets in the switchboard are terminals of the auxiliary outputs. Which output is wired to which socket depends on the installation. The outputs are controlled with `aux<N>State`, see ¹ for the output types that can be switched |
| Filtration pump on/off | `filtrationState` |
| Salt electrolysis | `salinity`, `chlorinatorOutputClPercent`, `targetChlorinatorOutputClPercent` |
| Water level | `poolWaterLevel`, `tankWaterLevel`, `waterRefill` |
| Temperatures | `poolTemp`, `temperatureIntoPool`, `outsideTemp`, `ambientTemperature` |

## Functions not available through the API

The following functions are intentionally not exposed:

* **Enabling and disabling pH and chlorine regulation.** The dosing control logic uses self-learning mechanisms that would be disturbed if an external system interrupted the dosing. The target values (`targetpHLevel`, `targetORPLevel`, `targetFreeClLevel`) can be set.
* **Time schedules.** The configuration of schedules must follow specific rules of the control logic, and its correct operation cannot be guaranteed if schedules are changed from outside.

The following functions are not available yet:

* **Alarms and error states.** The API does not provide access to them.

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
