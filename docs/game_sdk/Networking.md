# Networking

> danger
> The Discord Store is still in a beta period. All documentation and functionality can and will change.

Need a networking layer? Have a networking layer! This manager handles all things packets so you can get data from player to player and make your multiplayer...work. It:

- Functions as a connection-oriented, TCP-like API, but over UDP!
- Supports "reliable" and "unreliable" connections
  - Packets with loot in them always get there, but player positioning can be eventually consistent
- Features P2P-like connections, but routed through Discord's high-end server infrastructure
  - All the benefits of direct connections, without the IP leaks!
- Is encrypted!

An important note to make here is that our networking layer **is not peer-to-peer**. Discord has always promised that we will not leak your IP, and we promise to keep it that way. Though it seems like you are connected directly to another user, it routes through Discord's servers in the middle, ensuring both security and robust networking thanks to our servers.

## GetSessionId

Get the unique networking session id for the current user. Each user has a unique id for that session, used for connecting to that user.

Returns a `UInt64`.

###### Parameters

None

###### Example

```cs
var sid = networkManager.GetSessionId();
```

## Flush

Flushes the network. Run this at the end of your game's loop, once you've finished sending all you need to send. In Unity, for example, stick this in `LateUpdate()`.

Returns `void`.

###### Parameters

None

###### Example

```cs
OnLateUpdate()
{
  networkManager.Flush();
}
```

## OpenChannel

Opens an unreliable channel to the given session id on the given channel number. Use this type of channel for loss-tolerant data, like player positioning in the world.

Returns `void`.

###### Parameters

| name            | type   | description                     |
| --------------- | ------ | ------------------------------- |
| remoteSessionId | UInt64 | the session id to connect to    |
| channel         | byte   | the channel on which to connect |

###### Example

```cs
// In reality, you'd fetch this from their lobby metadata
var playerARemoteSessionId = 1234;
networkManager.OpenChannel(playerARemoteSessionId, 0);
```

## OpenReliableChannel

Opens a reliable channel to the given session id on the given channel number. Use this type of channel for data that _must_ get to the user, like loot drops!

Returns `void`.

###### Parameters

| name            | type   | description                     |
| --------------- | ------ | ------------------------------- |
| remoteSessionId | UInt64 | the session id to connect to    |
| channel         | byte   | the channel on which to connect |

###### Example

```cs
// In reality, you'd fetch this from their lobby metadata
var playerARemoteSessionId = 1234;
networkManager.OpenReliableChannel(playerARemoteSessionId, 1);
```

## SendMessage

Sends data to a given sessionid through the given channel.

Returns `void`.

###### Parameters

| name            | type   | description                     |
| --------------- | ------ | ------------------------------- |
| remoteSessionId | UInt64 | the session id to connect to    |
| channel         | byte   | the channel on which to connect |
| data            | byte[] | the data to send                |

###### Example

```cs
// In reality, you'd fetch this from their lobby metadata
var playerARemoteSessionId = 1234;

byte[] lootDrops = GameEngine.GetLootData();
networkManager.SendMessage(playerARemoteSessionId, 1, lootDrops);
```

## CloseChannel

Close the connection to a given remote seesion id on the given channel.

Returns `void`.

###### Parameters

| name            | type   | description                               |
| --------------- | ------ | ----------------------------------------- |
| remoteSessionId | UInt64 | the session id of the connection to close |
| channel         | byte   | the channel to close                      |

###### Example

```cs
// In reality, you'd fetch this from their lobby metadata
var playerARemoteSessionId = 1234;
void CloseChannel(UInt64 playerARemoteSessionId, byte channel);
Console.WriteLine("Connection to {0} closed", playerARemoteSessionId);
```

## OnMessage

Fires when you receive data from another user. This callback will only fire if you already have an open channel with the user sending you data. Make sure you're running `RunCallbacks()` in your game loop, or you'll never get data!

###### Parameters

| name            | type   | description                  |
| --------------- | ------ | ---------------------------- |
| senderSessionId | UInt64 | the session id of the sender |
| channel         | byte   | the channel it was sent on   |
| data            | byte[] | the data sent                |

###### Example

```cs
OnMessage += (senderSessionId, channel, data) =>
{
  var stringData = Encoding.UTF8.GetString(data);
  Console.WriteLine("Message from {0}: {1}", senderSessionId, stringData)
}
```

## Connecting to Each Other

This manager is built around the concept of channels between players. Player A opens a channel to Player B, and then Player B does the same to Player A. These two users can now send data back and forth to each other with `SendMessage()` and receive it with `OnMessage`.

In order to properly send and receive data between A and B, both users need to have **the same type of channel** open to each other **on the same channel number**. If A has Reliable Channel 4 open to B, B also needs Reliable Channel 4 open to A. There are many ways to keep track of this; utilizing metadata on the lobby is one great way.

###### Example: Storing Open Channels on Lobby Metadata

```cs
// In reality, you'd fetch this from their lobby metadata
var playerARemoteSessionId = 1234;

// Player B opens a connection to A
networkManager.OpenChannel(playerARemoteSessionId, 0);

// Create a lobby transaction to update metadata
var txn = lobbyManager.GetLobbyTransaction(lobby.id);

// Some struct made for this purpose
var metadata = new OpenChannel()
{
  owner = B.id,
  otherPerson = A.id,
  channel = 0,
  type = "reliable"
};

// Set the lobby metadata to note that B has an open channel to A
txn.SetMetadata("openChannels", JSON.stringify(metadata), (result) =>
{
  Debug.Log(result);
});

lobbyManager.UpdateLobby(lobby.id, txn, (result) =>
{
  Debug.Log(result);
})

// Player A can now listen for the OnLobbyUpdate callback and read the metadata
```

## Flush vs RunCallbacks

A quick note here may be helpful for the two functions that should be called continuously in your game loop: `discord.RunCallbacks()` and `networkManager.Flush()`. `RunCallbacks()` pumps the SDK's event loop, sending any newly-received data down the SDK tubes to your game. For this reason, you should call it at the beginning of your game loop; that way, any new data is handled immediately by callbacks you've registered. In Unity, for example, this goes in `Update()`.

`Flush()` is specific to the network manager. It actually _writes_ the packets out to the stream. You should call this at the _end_ of your game loop as a way of saying "OK, I'm done with networking stuff, go send all the stuff to people who need it". In Unity, for example, this goes in `LateUpdate()`.

## Example: Connecting to Another Player in a Lobby

```cs
var discord = new Discord.Discord(clientId, Discord.CreateFlags.Default);

// Join a lobby with another user in it
// Get their session id, and connect to them

var networkManager = discord.GetNetworkManager();
var lobbyManager = discord.GetLobbyManager();
var userManager = discord.GetUserManager();

var me;
var otherUserSessionId;

// Get yourself
userManager.GetCurrentUser((user) =>
{
  me = user;
});

// Connect to lobby with an id of 12345 and a secret of "password"
lobbyManager.Connect(12345, "password", (lobby) =>
{
  // Add our own session id to our lobby member metadata
  // So other users can get it to connect to us
  var txn = lobbyManager.CreateMemberTransaction();
  txn.SetMetadataString("sessionId", networkManager.GetSessionId());
  lobbyManager.UpdateMember(lobby.id, me.id, txn, (result) =>
  {
    // Who needs error handling anyway
    Console.WriteLine(result);
  }

  // Get the first member in the lobby, assuming someone is already there
  var member = lobbyManager.GetMemberUserId(lobby.id, 0);

  // Get their session id from their metadata, added previously
  otherUserSessionId = lobbyManager.GetMemberMetadata(lobby.id, member.id, "sessionId");
}

// Open an unreliable channel to the user on channel 0
// And a reliable one on channel 1
networkManager.OpenChannel(otherUserSessionId, 0);
networkManager.OpenReliableChannel(otherUserSessionId, 1);

// An important data packet from our game engine
byte[] data = GameEngine.GetImportantData();

if (isDataAboutPlayerLootDrops(data))
{
  // This is important and has to get there
  // Send over reliable channel
  networkManager.SendMessage(otherUserSessionId, 1, data);
}
else
{
    // This is eventually consistent data, like the player's position in the world
    // It can be sent over the unreliable channel
    networkManager.SendMessage(otherUserSessionId, 0, data);
}

// Done; ship it!
networkManager.Flush();
```
