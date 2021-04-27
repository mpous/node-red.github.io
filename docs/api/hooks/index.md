---
layout: docs-api
toc: toc-api-single.html
title: Hooks API
slug: hooks
---
The Hooks API provides a way to add custom code at specific points of the runtime.

Unlike events, hooks are inserted into the code path when something happens. They
can be used to modify the data being processed.

There are two groups of hooks: Message Routing hooks and Module Install hooks.

### Adding a hook

```javascript
RED.hooks.add(hookName, hookFunction)
```
 - `hookName` - the id of the hook with an optional label. For example, `"onSend"` and `"onSend.MyLabel"`.
   Only hooks with labels can be removed - it is a good idea to include a label for debugging
   and tracing purposes.
 - `hookFunction` - this is the function that will be called with the hook is triggered.
   ```javascript
   function(event [, done]) { ... }
   ```

The callback function is called with an `event` object - the specific details of the object
will depend on what hook has been triggered.

The function can accept an optional `done` callback that must be called once the
function has finished its work.

If the function does not take a second argument, then it must return its result
at the end of the function. The result *can* be a `Promise` that will resolve asynchronously.

### Removing a hook

```javascript
RED.hooks.remove(hookName)
```

Only hooks with labels can be removed.

To remove all hooks for a given label, `*.<label>` can be used.

### Message Routing hooks

![](./message-router-events.png)

#### onSend

 - `event`: An array of [SendEvent](#sendevent-object) objects

Triggered when a node calls `node.send()`. The messages inside these objects are
exactly what the node has passed to `node.send`. This means there could be
duplicate references to the same message object.

*See below regarding [async vs sync handlers](#async-vs-sync-handlers)*

Returning `false` will halt any further processing of this message.

#### preRoute

 - `event`: A [SendEvent](#sendevent-object) object

Triggered when an individual message is ready to be routed to its destination.

*See below regarding [async vs sync handlers](#async-vs-sync-handlers)*

Returning `false` will halt any further processing of this message.

#### preDeliver

 - `event`: A [SendEvent](#sendevent-object) object

Triggered when the local router has identified the node it is going to send this
message to. At this point, the message will have been cloned if needed.

Returning `false` will halt any further processing of this message.

#### postDeliver

 - `event`: A [SendEvent](#sendevent-object) object

Triggered after the message has been dispatched to be delivered asynchronously

#### onReceive

 - `event`: A [ReceiveEvent](#receiveevent-object) object

Triggered when a node is about to receive a message

#### postReceive

 - `event`: A [ReceiveEvent](#receiveevent-object) object

Triggered after the message has been given to the node's input handler.

#### onComplete

- `event`: A [CompleteEvent](#completeevent-object) object

Triggered when a node has completed with a message or logged an error

#### Event objects

##### SendEvent object

```json
{
    "msg": <message object>,
    "source": {
        "id": <node-id>,
        "node": <node-object>,
        "port": <index of port being sent on>,
    },
    "destination": {
        "id": <node-id>,
        "node": undefined,
    },
    "cloneMessage": true|false
}
```
##### ReceiveEvent object

```json
{
    "msg": <message object>,
    "destination": {
        "id": <node-id>,
        "node": <node-object>,
    }
}
```

##### CompleteEvent object

```json
{
    "msg": <message object>,
    "node": {
        "id": <node-id>,
        "node": <node-object>
    },
    "error": <error passed to done, otherwise, undefined>
}
```

#### Async vs Sync handlers

Due to the requirements of the messaging path, some hook handlers must complete
their work synchronously.

The `onSend` and `preRoute` hooks are triggered *before* any message cloning has
happened. If they complete asynchronously, execution will return to the calling
node *without* the message having been cloned. That will lead to unexpected issues
as the node may modify the message object before the original version has been delivered.

If `onSend` and `preRoute` hooks need to do asynchronous work before the event passes
on, and the `cloneMessage` property is set to `true`, then they *must* clone and
replace the message object in the event object. They *must* also set the `cloneMessage`
property to false to ensure no subsequent cloning happens for the message.

Subsequent hooks are called *after* the platform has done any required cloning, so
are free to act asynchronously or synchronously.

### Module Install hooks
*Added in Node-RED 2.x*

#### preInstall
*Added in Node-RED 2.x*

 - `event`: An [InstallEvent](#installevent-object) object

Triggered immediately before `npm install` is invoked to install a module.

Returning `false` will cause the built-in `npm install` to be skipped - on the
assumption that the handler has done all the work needed to install the module.

To customise the arguments passed to `npm install`, the handler can modify the
`event.args` property.

#### postInstall
*Added in Node-RED 2.x*

 - `event`: An [InstallEvent](#installevent-object) object

Triggered after `npm install` has completed successfully.

If the install fails for any reason, the postInstall hooks will not get triggered.


#### preUninstall
*Added in Node-RED 2.x*

 - `event`: An [UninstallEvent](#uninstallevent-object) object

Triggered immediately before `npm uninstall` is invoked to remove a module.

Returning `false` will cause the built-in `npm remove` to be skipped - on the
assumption that the handler has done all the work needed to remove the module.

To customise the arguments passed to `npm remove`, the handler can modify the
`event.args` property.

#### postUninstall
*Added in Node-RED 2.x*

 - `event`: An [UninstallEvent](#uninstallevent-object) object

Triggered after `npm remove` has completed successfully.

#### Event objects

##### InstallEvent object

```json
{
    "module": "<npm module name>",
    "version": "<version to be installed>",
    "url": "<optional url to install from>",
    "dir": "<directory to run the install in>",
    "isExisting": "<boolean> this is a module we already know about",
    "isUpgrade": "<boolean> this is an upgrade rather than new install",
    "args": [ "an array", "of the args" ,  "we will pass to npm"]
}
```

##### UninstallEvent object

```json
{
    "module": "<npm module name>",
    "dir": "<directory to run the uninstall in>",
    "args": [ "an array", "of the args" ,  "we will pass to npm"]
}
```
