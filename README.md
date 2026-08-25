# Technical Information for Modding Among Us
This space is for sharing technical information that may be useful for modding Among Us and playing mods on Innersloth's Among Us backend.
For Innersloth's policy on modding, please see: https://www.innersloth.com/among-us-mod-policy/
For inquiries about modding, please contact `modding@innersloth.com`

_Innersloth does not officially support mods and may change requirements or restrictions at anytime, especially if changes are required to increase security or protect players!_

## Classification of Mods
From the perspective of mods that use Innersloth's backend, there are two major categories of mods, which are treated differently in Innersloth's servers.

#### 1 - Host-only Mods
These mods require only the host's client to be modded to function, relying on host-authority of state to achieve modded functionality. While additional functionality is limited, accessibility to users without the mod installed is increased.

#### 2 - All-client Mods
These mods require all clients connected to the lobby to be modded with the same mod to function, providing more freedom to modify the game's behavior, at the cost of less accessibility to users.

## Mod Registration/Identification
By registering your mod with the Among Us game server that hosts your modded lobby, your modded lobby is granted special treatment with respect to state authority and validation.

### Host-only Mod Registration +25 Modded-Flag
_Applies to Host-only mods only_

Among Us clients contain an int32 networking version we call the protocol version. Adding `25` to this version enables a "host authority mode" for the lobby.

In "host authority mode", most Among Us gameplay features that have server authority is switched to host authority. This allows the host to hold power over the outcome of various gameplay features, enabling the ability to change the game's behavior.

#### Limitations
- Authority over all state is not granted to hosts such as with state related to user data or player protections
- While anti-cheat is relaxed, it is not guaranteed to be turned off in all cases
- Rate-limiting and other security protections may still apply

### All-client Mod Client Identification
_Applies to all-client mods only_

Innersloth is happy to present the Among Us Modded Client Identification (AU MCI) tools. The AU MCI tooling gives the modding community the ability to (via opt-in) declare their mods as a part of the game hosting processing and get access to features like:
- A Lobby search that exclusively finds games running the same mod
- Lobbies that non-modded clients can't join
- Unique exemptions from anti-cheat measures

While we do not officially provide technical support for mods (see our Mod Policy here: https://www.innersloth.com/among-us-mod-policy/), we want to give the modding community better transparency and insight, while also bolstering our anti-cheat measures in non-modded games. 


## How to register an All-client Mod

### Overview
The AU MCI is made up of three components:
- A self-assigned Mod GUID
- A dedicated path to host modded games with your own Mod GUID
- Matchmaking Filters for filtering lobbies against your own Mod GUID

_The code samples below are provided in C# and assume a working familiarity with modding the Among Us client code_

#### Mod GUID/UUID
The self-assigned Mod GUID is an identification tool for a modder to uniquely identify their game from other mods. Examples would look like: 
```501f405a-7d89-4505-b5a8-f4200c10d625
8afb3cb8-ad33-4fe5-a125-e4741ff116fe
bc634373-dd53-453b-8711-86e4ba3d0c32
72784e1c-13b0-4118-bf01-144b9371c6c4
6ba99692-6f5e-4e46-9774-77704cb595b6
728b931b-badf-46b0-9d7c-90a6bc6ddf0a
b887635f-e35e-42a3-b950-0ebf2b05f9f8
73f2dd39-106c-4422-aa51-0f89f12b42cb
```
These are generic V4 UUIDs, and you can acquire a randomly generated Mod GUID for yourself online at sites like: https://www.uuidgenerator.net/ This GUID is how we distinguish your mod from other mods, so *it's important to use the same one in the other steps below*. 

#### Registration methods
Ultimately, registration requires understanding of how to modify Among Us, by finding where in the Among Us codebase to inject or change functionality. Innersloth does not provide information on how to modify the Among Us application. However, below we provide information on how Mod registration can be achieved.

### Built-in Registration Helpers (Among Us 18.0+)
From community feedback, we understood that inline can create difficulties in adding mod registration to a mod. We decided to build in mod registration functionality directly into the client, providing an easier way to implement.

We've added static class called `CurrentModRegistration`.

The class has a static string named `ModRegistrationGuidString`.

By default, a null or empty string will be interpreted by the Among Us code base to have no mod registration, and will use the normal HostGame methods and will not add mod filtration to matchmaking requests (see Advanced Technical Details below).

However, if this string is value is modded to your mod's GUID, it will automatically apply to host game functionality, and host your game w/ the expected HostModdedGame tag, including the specified GUID. Furthermore, the matchmaking filtration code will detect this change and automatically add a mod filter using the same GUID.

With this feature, it should be possible to registration a mod GUID with greater ease compared with previous version of Among Us.

#### Additional Details
The `bool CurrentModRegistration.TryGetModRegistrationGuid(out Guid guid)` static method is the method which other systems use to detect a mod GUID. It includes null/string empty check as well as GUID parsing to ensure that the GUID string is a valid GUID.

Mods that may want to switch between registered GUIDs may be able to leverage `ModRegistrationGuidString` or `TryGetModRegistrationGuid` to dynamically switch mod GUIDs before creating or searching for a lobby.

### Advanced Technical Details
In this section, we provide more technical details in how registration and filtration works.

#### Hosting Modded Games
Hosting a modded game requires the use of the new `Tags.HostModdedGame` (byte value of 25) tag packaged in the client. In the `InnerNetClient`'s `HostGame` method, modify the line

`msg.StartMessage(Tags.HostGame);`

to 

`msg.StartMessage(Tags.HostModdedGame);`

Next, append your Mod GUID to the host game message:
  ```
  // Standard HostGame method body
  MessageWriter msg = MessageWriter.Get(SendOption.Reliable);
  msg.StartMessage(Tags.HostModdedGame);
  msg.WriteBytesAndSize(this.gameOptionsFactory.ToBytes(settings, AprilFoolsMode.IsAprilFoolsModeToggledOn));
  msg.Write(CrossplayMode.GetCrossplayFlags());
  filterOpts.Serialize(msg);
  
  // Serializing your GUID as bytes in the message 
  bool guidSucceeded = Guid.TryParse("316e7f61-f150-4ac0-b2cd-7f3cc7225963", out Guid guid); // example GUID
  if (!guidSucceeded)
  {
      Debug.LogError("Whoa guid failed to generate");
      return;
  }
  msg.Write(guid.ToByteArray());

  // Standard HostGame method
  msg.EndMessage();
  this.SendOrDisconnect(msg);
  msg.Recycle();
  ```

This allows our game server and matchmakers to mark all the games hosted by your mod as using the same mod and make them available for matchmaking. At this point, they will be excluded from the normal matchmaking pool and exempted from the anti-cheat system as well.

#### Filtering against your own modded lobbies
Modifying the Matchmaking filter system will allow us to serve games with your Mod GUID in the regular Find Game lobby search screen. 

First, you will need to define a matchmaking filter to package with your game searches. The important part is that the Mod GUID you selected above is available in the `AcceptedValues` property and that the `FilterType` corresponds to the string value "mod":
```
    [Serializable]
    public class ModFilter : ISubFilter
    {
        public Guid AcceptedValues;
        public string FilterType { get; } = "mod"; 
    } 
```

Then, in the matchmaking flow, add a filter containing your GUID to the existing set of filters:
```
        Guid guid = new Guid("316e7f61-f150-4ac0-b2cd-7f3cc7225963");

        filterSet.Filters.Add(new GameFilter("mod",
            new ModFilter()
            {
                AcceptedValues = guid,
            }));
```

By adding these components, the Find Game screen will tell our matchmakers that you're a modded client with your Mod GUID, looking for other games running this same mod.

Thank you for helping us make Among Us bigger and better! If you have any questions or concerns, please refer to our mod policy or reach out to us at `modding@innersloth.com`.

### FAQ 
*Does this replace the +25 Modded Flag?*

No, the +25 modded flag is a separate option that changes some server authoritative logic to host authoritative logic, popular for host-only mods.

*Can the +25 Modded Flag and Mod GUID be used together?*
Yes, the AU MCI mod GUID registration and the +25 modded flag can be used in combination, depending on the mods needs.

# Mod Features

## Host-only Mods

### Chat commands
We've added the ability for non-host players in host-only mods to directly send a chat command to the host. This allows the host to implement roles with special behaviors using chat as a way to use special abilities.

Any mod with the +25 modded flag activates this feature on the lobby they are playing on. When active, when non-hosts send chat messages that start with `/cmd`, the chat message will only be sent to the host and will not be sent to any other player in the lobby.

#### Usage example:
A host-only mod with a role that can guess the impostor might use a format such as:

`/cmd guess IsThisTheImpostor 1`

The modded host can parse this message to interpret the command `guess` and the name of the suspected impostor `IsThisTheImpostor 1`, then can use a `GameDataTo` response to send a chat privately to only the guessing player.

### Packed GameDataTo messages
We've added the ability for GameDataTo messages to be packed together for host-only mods. Each packed GameDataTo message in the packet can be directed to different remote players in the lobby.

Host-only mods have devised methods to create new functionality that relies on desynchronizing the state that each remote player experiences. These methodologies often require a burst of packets to update each remote player in a fan out to n-1 players for a lobby with n players.

We need to be mindful about packet send rates to maintain security and performance of the Among Us game servers. As such, mods should always do as much as possible to minimize packet send rate for non-vanilla Among Us functionality. See more information below on Packet Send Rates for strategies to minimize send rate beyond using GameDataTo packing.

To use GameDataTo packing, use the top level message tag `Tags.PackedGameDataTo` which has a byte value of `26`. The message requires the GameId. Pseudocode example is as follows:

```
int currentGameId = AmongUsClient.Instance.GameId;
MessageWriter msg = MessageWriter.Get(SendOption.Reliable);
msg.StartMessage(Tags.PackedGameDataTo);
msg.WritePacked(currentGameId);

foreach (submsg to pack)
{
    msg.StartMessage(Tags.GameDataTo);
    msg.Write(currentGameId);
    msg.WritePacked(submsg target ClientId);
    msg.StartMessage((byte)GameDataTypes.RpcFlag or DataFlag);
    // serialize submsg contents
    msg.EndMessage(); // Rpc/DataFlag
    msg.EndMessage(); // GameDataTo
}

msg.EndMessage(); // PackedGameDataTo
```

Requirements:
1. Can only pack GameDataTo Tag types together
2. Each GameDataTo message must contain the correct game id
3. Full MessageWriter packet size, including header must be less than or equal to 1200 bytes

# Packet Send Rates
Packet send rates are an important consideration for multiplayer games for performance and security. All clients that connect to Innersloth's Among Us GameServers are required to follow any required limitations on packet send rates and sizes.

When developing a mod for Among Us, you must take packet send rates and packet sizes in consideration.

## Send rates
Innersloth is still in the process of determining exact send rate requirements for mods. While it is likely we can provide some extra room for mods, security comes first and we may need to apply strict rate limits to prevent disruption to the playerbase.

If you're an Among Us mod developer, you must take care to minimize send rates. If there's a functionality that you think can only be achieved with a burst of messages, please contact `modding@innersloth.com` to explain your use case.

### Strategies to reduce send rate
Here are some strategies that can be applied to reduce send rate:
1. Use the GameDataTo packing feature (see above)
2. Split messages between Reliable and Unreliable channels when possible
3. Spread packets across multiple frames or seconds
4. Utilize built in rate limiting queues and streams (see below)

### Built in rate limit queues
1 - RPCs
Non-competitive RPCs use a queue for optimized send rate packing:

```
// Reliable example
RpcSetScannerMessage rpc = new RpcSetScannerMessage(this.NetId, value, cnt);
AmongUsClient.Instance.LateBroadcastReliableMessage(rpc);

// Unreliable example
RpcPlayAnimationMessage rpc = new RpcPlayAnimationMessage(this.NetId, animType);
AmongUsClient.Instance.LateBroadcastUnreliableMessage(rpc);
```

2 - DataFlags
All DataFlag messages, which update the state of an InnerNetObject, are sent using message streams and a dirty flag pattern. Updating a streamed objects data and setting it to dirty will induce the streamed object system to send the update to all players.

InnerNetObjects generally auto-set their dirty state using `this.SetDirtyBit(1);`.

## Packet sizes
The maximum `MessageReader` messasge size allowed (header included) is 1200 bytes.

### Strategies to control packet sizes
A simple strategy to control packet size is to use a queue and pack messages until adding an additional message would push the size of the packet beyond the capacity limit. This strategy can be used with rate limiting strategies to control both the size and rate of outgoing packets.
