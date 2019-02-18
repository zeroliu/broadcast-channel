
<p align="center">
  <a href="https://github.com/pubkey/broadcast-channel">
    <img src="https://assets-cdn.github.com/images/icons/emoji/unicode/1f4e1.png" width="150px" />
  </a>
</p>

<h1 align="center">BroadcastChannel</h1>
<p align="center">
  <strong>A BroadcastChannel that works in old browsers, new browsers, WebWorkers and NodeJs</strong>
  <br/>
  <span>+ LeaderElection over the channels</span>
</p>

<p align="center">
    <a alt="travis" href="https://travis-ci.org/pubkey/broadcast-channel">
        <img src="https://travis-ci.org/pubkey/broadcast-channel.svg?branch=master" /></a>
    <a href="https://twitter.com/pubkeypubkey">
        <img src="https://img.shields.io/twitter/follow/pubkeypubkey.svg?style=social&logo=twitter"
            alt="follow on Twitter"></a>
</p>

<br/>

* * *

A BroadcastChannel allows simple communication between browsing contexts with the same origin or different NodeJs processes.

This implementation works with old browsers, new browsers, WebWorkers and NodeJs. You can use it to send messages between multiple browser-tabs, iframes, WebWorkers and NodeJs-processes.

This behaves similar to the [BroadcastChannel-API](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API) which is currently not featured in all browsers.

## Using the BroadcastChannel

This API behaves similar to the [javascript-standard](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API).

```bash
npm install --save broadcast-channel
```

#### Create a channel in one tab/process and send a message.

```js
const BroadcastChannel = require('broadcast-channel');
const channel = new BroadcastChannel('foobar');
channel.postMessage('I am not alone');
```

#### Create a channel with the same name in another tab/process and recieve messages.

```js
const BroadcastChannel = require('broadcast-channel');
const channel = new BroadcastChannel('foobar');
channel.onmessage = msg => console.dir(msg);
// > 'I am not alone'
```


#### Add and remove multiple eventlisteners

```js
const BroadcastChannel = require('broadcast-channel');
const channel = new BroadcastChannel('foobar');

const handler = msg => console.log(msg);
channel.addEventListener('message', handler);

// remove it
channel.removeEventListener('message', handler);
```

#### Close the channel if you do not need it anymore.

```js
channel.close();
```

#### Set options when creating a channel (optional):

```js
const options = {
    type: 'localstorage', // (optional) enforce a type, oneOf['native', 'idb', 'localstorage', 'node']
    webWorkerSupport: true; // (optional) set this to false if you know that your channel will never be used in a WebWorker (increases performance)
};
const channel = new BroadcastChannel('foobar', options);
```

#### Create a typed channel in typescript:

```typescript
import BroadcastChannel from 'broadcast-channel';
declare type Message = {
  foo: string;
};
const channel: BroadcastChannel<Message> = new BroadcastChannel('foobar');
channel.postMessage({
  foo: 'bar'
});
```


#### Clear tmp-folder:
When used in NodeJs, the BroadcastChannel will communicate with other processes over filesystem based sockets.
When you create a huge amount of channels, like you would do when running unit tests, you might get problems because there are too many folders in the tmp-directory. Calling `BroadcastChannel.clearNodeFolder()` will clear the tmp-folder and it is recommended to run this at the beginning of your test-suite.

```jest
beforeAll(async () => {
  const hasRun = await BroadcastChannel.clearNodeFolder();
  console.log(hasRun); // > true on NodeJs, false on Browsers
})
```

```Mocha
before(async () => {
  const hasRun = await BroadcastChannel.clearNodeFolder();
  console.log(hasRun); // > true on NodeJs, false on Browsers
})
```


## Methods:

Depending in which environment this is used, a proper method is automatically selected to ensure it always works.

| Method           | Used in                                                         | Description                                                                                                                                             |
| ---------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Native**       | [Modern Browsers](https://caniuse.com/broadcastchannel)         | If the browser supports the BroadcastChannel-API, this method will be used because it is the fastest                                                    |
| **IndexedDB**    | [Browsers with WebWorkers](https://caniuse.com/#feat=indexeddb) | If there is no native BroadcastChannel support, the IndexedDB method is used because it supports messaging between browser-tabs, iframes and WebWorkers |
| **LocalStorage** | [Older Browsers](https://caniuse.com/#feat=namevalue-storage)   | In older browsers that do not support IndexedDb, a localstorage-method is used                                                                          |
| **Sockets**      | NodeJs                                                          | In NodeJs the communication is handled by sockets that send each other messages                                                                         |



## Using the LeaderElection

This module also comes with a leader-election which can be used so elect a leader between different BroadcastChannels.
For example if you have a stable connection from the frontend to your server, you can use the LeaderElection to save server-side performance by only connecting once, even if the user has opened your website in multiple tabs.

In this example the leader is marked with the crown ♛:
![leader-election.gif](docs/files/leader-election.gif)


Create a channel and an elector.

```js
const BroadcastChannel = require('broadcast-channel');
const LeaderElection = require('broadcast-channel/leader-election');
const channel = new BroadcastChannel('foobar');
const elector = LeaderElection.create(channel);
```

Wait until the elector becomes leader.

```js
const LeaderElection = require('broadcast-channel/leader-election');
const elector = LeaderElection.create(channel);
elector.awaitLeadership().then(()=> {
  console.log('this tab is now leader');
})
```

If more than one tab is becoming leader adjust `LeaderElectionOptions` configuration.

```js
const LeaderElection = require('broadcast-channel/leader-election');
const elector = LeaderElection.create(channel, {
  fallbackInterval: 2000, // optional configuration for how often will renegotiation for leader occur
  responseTime: 1000, // optional configuration for how long will instances have to respond
});
elector.awaitLeadership().then(()=> {
  console.log('this tab is now leader');
})
```

Let the leader die. (automatically happens if the tab is closed or the process exits).

```js
const elector = LeaderElection.create(channel);
await elector.die();
```



## What this is

This module is optimised for:

- **low latency**: When you postMessage on one channel, it should take as low as possible time until other channels recieve the message.
- **lossless**: When you send a message, it should be impossible that the message is lost before other channels recieved it
- **low idle workload**: During the time when no messages are send, there should be a low processor footprint.

## What this is not

-   This is not a polyfill. Do not set this module to `window.BroadcastChannel`. This implementation behaves similiar to the [BroadcastChannel-Standard](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API) with these limitations:
    - You can only send data that can be `JSON.stringify`-ed.
    - While the offical API emits [onmessage-events](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel/onmessage), this module directly emitts the data which was posted
-   This is not a replacement for a message queue. If you use this in NodeJs and want send more than 50 messages per second, you should use proper [IPC-Tooling](https://en.wikipedia.org/wiki/Message_queue)


## Browser Support
I have tested this in all browsers that I could find. For ie8 and ie9 you must transpile the code before you can use this. If you want to know if this works with your browser, [open the demo page](https://pubkey.github.io/broadcast-channel/).

## Thanks
Thanks to [Hemanth.HM](https://github.com/hemanth) for the module name.
