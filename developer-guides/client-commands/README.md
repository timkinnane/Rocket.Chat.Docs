# Client Commands

Client Commands is a collection used to send _commands_ from the server to the client. Subscriptions only receive the documents targeted at the user that subscribed to the publication `clientCommands`.

The main goal is to ease the process of managing bot accounts, so its intended use is to send low-level directives to be interpreted by the client and not the user, such as clearing cache at the SDK or Adapter level or sending a `heartbeat` clientCommand that is instantly responded, in order to check aliveness.

## Specification

### `RocketChat.sendClientCommand(user, command)`

Function used to send a ClientCommand to a client that is subscribed to the `clientCommands` publication. It can only be called on server side.

- Parameter `user`: Object containing the properties `_id` and `username` of the user that should receive the clientCommand.
- Parameter `command`: Object containing an attribute called `key` that is an unique identifier of the clientCommand. It can also have other attributes needed to interpret the clientCommand.
- Returns a Promise that resolves with the full document or rejects with a Meteor.Error. If the promise is rejected, you can access the inserted document via `err.details.command`.

Then a document with the following structure will be inserted into the ClientCommands collection:

```
{
  _id: Random ID of the ClientCommand document,
  u: User object described above,
  cmd: Command object described above,
  ts: Timestamp of when the ClientCommand was created
}
```

Then it will wait for any changes in the inserted documented, resolving with the full changed object.

If no changes happen in 5 seconds, the promise will be rejected.

### `Meteor.methods({ replyClientCommand(commandId, response) })`

This method simply updates the ClientCommand with `_id: commandId` setting the `response` attribute with the `response` parameter.

## How to Use

### Sending a ClientCommand

To send a command to a client being used by a specific user, you have to call the `sendClientCommand` method, which has `user` and `command` as its parameters.

Example call, let's suppose the adapter used by `salesBot` and implemented by RC can have its cache cleared by admins:

```
const user = {
  _id: 'Xklds3ok4pc',
  username: 'salesBot'
};

const command = {
  key: 'clearCache'
}

try {
  const response = await RocketChat.sendClientCommand(user, command);
  if (response.success) {
    // OK
  } else {
    // NOT OK
  }
} catch(err) {
  handleTimeout(err);
}
```

A document was inserted in the ClientCommands collection, subscribed by the logged in user in its client (the adapter that can have its cache cleared).

In that example, if the client responds in less than 5 seconds (whether it succeeded or not), the clientCommand will be replied to and then one of the branches will be executed.

### Responding to a ClientCommand

To responde to a ClientCommand, you must first subscribe to the `clientCommands` publication and watch for any changes or insertions in the collection.

When you detect a new `clientCommand`, you should handle it according to its `key`, accessible via `clientCommand.cmd.key`.

Continuing the example above, the adapter (the client) could have something like this implemented:

```
if (command.cmd.key === 'clearCache') {
  try {
    await clearCache();
    callRCMethod('replyClientCommand', command._id, { success: true });
  } catch (err) {
    callRCMethod('replyClientCommand', command._id, { success: false });
  };
}
```

In that example, if clearCache resolves or rejects in less than 5 seconds, the clientCommand will be replied to and then the code that asked for the cache to be cleared will handle the success of the clientCommand. Otherwise, the sender of the clientCommand will declare a timeout and ignore any subsequent responses.
