# XRPL WebSocket Client [![npm version](https://badge.fury.io/js/xrpl-client.svg)](https://www.npmjs.com/xrpl-client) [![GitHub Actions NodeJS status](https://github.com/XRPL-Labs/xrpl-client/workflows/NodeJS/badge.svg?branch=main)](https://github.com/XRPL-Labs/xrpl-client/actions) [![CDNJS Browserified](https://img.shields.io/badge/cdnjs-browserified-blue)](https://cdn.jsdelivr.net/npm/xrpl-client/dist/browser.js) [![CDNJS Browserified Minified](https://img.shields.io/badge/cdnjs-minified-orange)](https://cdn.jsdelivr.net/npm/xrpl-client/dist/browser.min.js)

### XRP Ledger WebSocket Client, npm: `xrpl-client`

Auto reconnecting, buffering, subscription remembering XRP Ledger WebSocket client. For in node & the browser.

This client implements a check for a working XRPL connection: the WebSocket being simply online isn't enough to satisfy the online / offline detection of this lib. After connecting, this lib. will issue a `server_info` command to the other connected node. Only if a valid response is retrieved the connection will be marked as online.

#### Constructor & options

A client connection can be constructed with the exported `XrplClient` class:

```typescript
import { XrplClient } from "xrpl-client";
const client = new XrplClient();
// ^^ No arguments: defaults to one endpoint: wss://xrplcluster.com
// with `maxConnectionAttempts` option `null` (try forever)
```

If no argument is provided, the default endpoint this lib. will connect to is [`wss://xrplcluster.com`](https://xrplcluster.com), options will have their default values outlined below. Alternatively, two arguments can be provided:

###### Arguments

1. (string | array) The WebSocket endpoint to connect to (e.g. your own node) as a `string`, or an array (`string[]`) with multiple endpoints used in specified order
2. (object) Global options (type: WsClientOptions)

###### Options

Available options are:

- `assumeOfflineAfterSeconds`, `Number` » default **30**, this setting will check if the XRPL node on the other end of the connection is alive and sending regular `server_info` responses (this lib. queries for them). After the timeout, the lib. will disconnect from the node and try to reconnect.
- `maxConnectionAttempts`, `Number` | `null` » default **null** in case of one endpoint, or **5** if an array with endpoints is provided, if (when initially connecting or reconnecting) no (new) connection could be setup in this attempts (see: `connectAttemptTimeoutSeconds` per call) consider the connection dead. Cancel all connect/reconnect attempts, clear the command buffer. An error will be thrown.
- `connectAttemptTimeoutSeconds`, `Number` » default **4**, this setting is the max. delay between reconnect attempts, if no connection could be setup to the XRPL node. A backoff starting at one second, growing with 20% per attempt until this value is reached will be used.

Sample with custom nodes & options:

```typescript
import { XrplClient } from "xrpl-client";
const client = new XrplClient(
  ["ws://localhost:1337", "wss://xrplcluster.com"],
  {
    assumeOfflineAfterSeconds: 15,
    maxConnectionAttempts: 4,
    connectAttemptTimeoutSeconds: 2,
  }
);
```

In this example this lib. will:
- First try to connect to the node running at localhost and after `4` reconnect attempts it'll try the next node (xrplcluster.com)
- Once connected, if no new ledger information has been received from the node after 15 seconds (at any given moment) the client/connection/endpoint is considered no longer connected (offline) and the connection sequence will start again, starting with max. 4 attempts to the node running at localhost.

#### Methods:

- `send({ command: "..."}, {SendOptions})` » `Promise<AnyJson | CallResponse>` » Send a `comand` to the connected XRPL node.
- `ready()` » `Promise<self>` » fires when you're fully connected. While the `state` event (and `getState()` method) only return the WebSocket online state, `ready()` will only return (async) if the first ledger data has been received and the last ledger index is known.
- `getState()` » `ConnectionState` » Get the connection, connectivity & server state (e.g. fees, reserves).
- close() » `void` » Fully close the entire object (can't be used again).

#### Send options

The `send({ comand: "..." })` method allows you to set these options (second argument, object):

- `timeoutSeconds`, `Boolean` » The returned Promise will be rejected if a response hasn't been received within this amount of seconds. This timeout starts when the command is issued by your code, no matter the connection state (online or offline, possibly waiting for a connecftion)
- `timeoutStartsWhenOnline`, `Number` » The timeout (see `timeoutSeconds`) will start when the connection has been marked online (WebSocket connected, `server_info` received from the XRPL node), so when your command has been issued by this lib. to the XRPL node on the other end of the connection.
- `sendIfNotReady`, `Boolean` » Your commands will be sent to the XRPL node on the other end of the connection only when the connection has been marked online (WebSocket connected, `server_info` received from the XRPL node). Adding this option (`true`) will send your commands _after_ the WebSocket has been connected, but possibly _before_ a valid `server_info` response has been received by the XRPL node connected to.
- `noReplayAfterReconnect`, `Boolean` » When adding a subscription (resulting in async. updates) like a `subscribe` or `path_find` command, when reconnected your subscription commands will automaticaly replay to the newly connected node. Providing a `false` to this option will prevent your commands from being replayed when reconnected.

#### Events emitted:

- `state` (the state of the connection changed from online to offline or vice versa)
- `message` (all messages, even if duplicate of the ones below)
- `ledger` (a ledger closed)
- `path` (async `path_find` response)
- `transaction`
- `validation`
- `retry` (new connection attempt)
- `close` (upstream closed the connection)
- `reconnect` (reconnecting, after connected: `state`)

#### Connection flow events:

- `retry` - Same node, new connection attempt (attempt timed out)
- `nodeswitch` » string (node) - Switched to a new node
- `online` » Now conneted to an XRPL node, use `.getState()` for more info
- `offline` » Was online, but the connection is gone
- `round` » Tried all nodes, retry the first one

Let's say you have two dead endpoints and a third one that works, then your connection is lost and you switch to the fourth one. The event sequence would look like this:

```
1. retry » 2. retry » 3. retry » 4. nodeswitch
5. retry » 6. retry » 7. retry » 8. nodeswitch
9. online
10. offline
11. retry » 12. retry » 13. retry » 14. nodeswitch
15. online
```

### Syntax

```typescript
import { XrplClient } from "xrpl-client";
const client = new XrplClient("wss://xrplcluster.com");

// await client.ready();

const serverInfo = await client.send({ command: "server_info" });
console.log({ serverInfo });

client.on("ledger", (ledger) => {
  console.log("Ledger", ledger);
});
```

### Use in the browser

You can clone this repository and run:

- `npm run install` to install dependencies
- `npm run build` to build the source code
- `npm run browserify` to browserify this lib.

Now the `dist/browser.js` file will exist, for you to use in a browser.

Alternatively you can get a [prebuilt](https://cdn.jsdelivr.net/npm/xrpl-client/dist/browser.js) / [prebuilt & minified](https://cdn.jsdelivr.net/npm/xrpl-client/dist/browser.min.js) version from Github.

Sample: [https://jsfiddle.net/WietseWind/p4cd37hf](https://jsfiddle.net/WietseWind/p4cd37hf/)

### Migrating from `rippled-ws-client`

1. The constructor doesn't return a promise with the connection: the constructed object passes on your messages. So if you need to wait for a live connection: use `await TheObject.ready()` and then refer to `TheObject`:

```javascript
// Old:
//    new RippledWsClient('wss://testnet.xrpl-labs.com').then(Connection => { ... })
// New:
const Connection = new RippledWsClient('wss://testnet.xrpl-labs.com')
Connection.ready().then(() => {
```

3. When used in combination with `rippled-ws-client-sign` ([please use `xrpl-accountlib` instead!](https://www.npmjs.com/package/xrpl-accountlib)) you need to wrap the class:

```javascript
class RippledWsClient extends XrplClient {} // Then use RippledWsClient
```
