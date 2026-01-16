# Technical Information for Modding Among Us
This space is for sharing technical information that may be useful for modding Among Us and playing mods on Innersloth's Among Us backend.
For Innersloth's policy on modding, please see: https://www.innersloth.com/among-us-mod-policy/
For inquiries about modding, please contact `modding@innersloth.com`

_Innersloth does not officially support mods and may change requirements or restrictions at anytime, especially if changes are required to increase security or protect players!_

## Classification of Mods
From the perspective of mods that use Innersloth's backend, there are two major categories of mods that are treated differently in Innersloth's servers.

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

#### Technical Summary
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

#### FAQ 
*Does this replace the +25 Modded Flag?*

No, the +25 modded flag is a separate option that changes some server authoritative logic to host authoritative logic, popular for host-only mods.
Both AU MCI and the +25 modded flag can be used in combination, depending on the mods needs.

# Mod Features

## Host-only Mods

### Chat commands
We've added the ability for non-host players in host-only mods to directly send a chat command to the host. This allows the host to implement roles with special behaviors using chat as a way to use special abilities.

Any mod with the +25 modded flag activates this feature on the lobby they are playing on. When active, when non-hosts send chat messages that start with `/cmd`, the chat message will only be sent to the host and will not be sent to any other player in the lobby.

#### Usage example:
A host-only mod with a role that can guess the impostor might use a format such as:

`/cmd guess IsThisTheImpostor 1`

The modded host can parse this message to interpret the command `guess` and the name of the suspected impostor `IsThisTheImpostor 1`, then can use a `GameDataTo` response to send a chat privately to only the guessing player.
