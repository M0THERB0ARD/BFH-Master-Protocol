# Battlefield Heroes master server protocol specification

This document provides a technical specification for the protocols that are used to communicate between the master server and the game client, and the master server and the game server. 

## Document version

Because this protocol is proprietary, not all details are available yet. This document may be updated as more details are reverse-engineered, mistakes are corrected, or ambiguities are clarified.
If you have any improvements you can make to this document, please send a pull request at https://github.com/M0THERB0ARD/BFH-Master-Protocol

|Version |Date      |Note           |
|--------|----------|---------------|
|1.0     |2017-11-04|Initial version|

## General infrastructure overview

Battlefield Heroes has a network structure similar to many other online games. It is based on previous games that also used the Refractor 2 game engine, such as Battlefield 2 or Battlefield 2142.
The general stack consists of the following components:
1. Game client: the front-end software that runs on the player's computer. Consists mainly of a graphical userinterface and some game-logic.
2. Game server: the back-end server that acts as a central game coordinator for the players in a match. Consists mainly of game logic and connections to game clients.
3. Master server: the back-end server that stores player and server data and does match-making. This server provides persistance in between matches.

This specification provides details on the communication between the game client and master server, and the game server and the master server. This documents does not specify the protocol between game server and game client.

## Master server overview

The master server has 3 main components:
1. Two FESL servers: a message based protocol server that handles authentication, quering account info, ...
2. The Magma server: a HTTPS based server for more account info
3. Two Theater servers: a message based protocol server that handles querying, joining, leaving, ... of game servers and clients

A game client will first connect to the FESL server, then the HTTP server, then the Theater server and finally the game server.

## Intermission: getting the game and server to start

In order to test and debug an implementation of the master server protocol, a working copy of the game client and server is required. Starting the game client can be done from command line with the following arguments:

`bfheroes.exe +sessionId ASessionIdAcceptedByTheServer +magma 0 +punkbuster 0 +developer 1`

The game server can be started using the following arguments:

`BFHeroes_w32ded.exe +key KeyToStartServer +eaAccountName TheAccountUsername +eaAccountPassword TheAccountPassword +soldierName TheServerName`

Whether or not the values for these arguments are valid depends on the implementation of the master server.
If the game server interface has the text "Online" at the top, it has successfully connected to the master server.
For more information, see the FESL chapter.

## FESL

### TLS
On startup, both the game client and the game server will first connect to their respective, seperate FESL server. 
The address and port of the FESL server is baked into the game client/server executable.
Known offsets of the FESL server address are:

|Version       |Product     |Offset     |
|--------------|------------|-----------|
|1.46.222034.0 |Game client |0x00951EA4 |
|1.42.217478.0 |Game server |0x0067329B |

The default value is "bfwest-server.fesl.ea.com".
The default port is 18270 for the game client and 18051 for the game server.

Communication over this connection is encrypted using TLS. By default, the game client/server checks the FESL server TLS certificate and disconnects if it does not match a preset EA certificate.
This check can be disabled using a patch to executable. (See Appendix: "FESL certificate patch")
After the patch, the game client/server will accept more but not all certificates.
The following OpenSSL command generates certificates and key files that are accepted by a patched game client/server:

`openssl.exe req -x509 -nodes -newkey rsa:1024 -keyout key.pem -out cert.pem -days 365 -sha1 -subj /C=US`

During the TLS handshake, both parties agree on a cipher suite and SSL version. Known good values for these are TLS_RSA_WITH_RC4_128_SHA and SSL 3.0 respectively.

### Packet structure
After the TLS handshake, FESL messages are exchanged over the encrypted line. 
The format for these messages is as follows:

|Offset (bytes) |Length (bytes)     |Data type                          |Field name    |
|---------------|-------------------|-----------------------------------|--------------|
|0x0            |4                  |ASCII string (no null terminator)  |Type          |
|0x4            |4                  |32-bit big-endian unsigned integer |ID            |
|0x8            |4                  |32-bit big-endian unsigned integer |Packet length |
|0xC            |Packet length - 12 |ASCII string (no null terminator)  |FESLData      |

The FESLData field is a key-value map where each pair is seperated by a newline (\n), and the key and value are seperated by '='.
For example:
```
Key1=Value1
Key2=Value2
```

### Message types
One of the keys in the FESLData key-value store is 'TXN'. This entry determines the message type.
Depending on the message type, and whether the message is to or from the FESL server, other fields may be present in the FESLData.
Response packets are always sent with the same Type and ID values as the query packet.

#### TXN = Hello, game client/server => FESL server
This is the first packet that is sent when a FESL connection is made.

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|SDKVersion                |5.0.0.0.0                  |                               |
|clientPlatform            |PC                         |                               |
|clientString              |bfwest-pc                  |                               |
|clientType                |server                     |                               |
|clientVersion             |1.46.222034                |                               |
|locale                    |en_US                      |                               |
|sku                       |125170                     |                               |
|protocolVersion           |2.0                        |                               |
|fragmentSize              |8096                       |                               |

#### TXN = Hello, FESL server => game client/server

|Key                       |Example value              |Note                                             |
|--------------------------|---------------------------|-------------------------------------------------|
|domainPartition.domain    |eagames                    |                                                 |
|domainPartition.subDomain |bfwest-server              |bfwest-server if the connected party is a server.|
|                          |                           |bfwest-dedicated otherwise.                      |
|curTime                   |Nov-02-2017 22:29:00 UTC   |                                                 |
|activityTimeoutSecs       |3600                       |                                                 |
|messengerIp               |messaging.ea.com           |This server is not required to play the game.    |
|messengerPort             |13505                      |                                                 |
|theaterIp                 |bfwest-pc.theater.ea.com   |                                                 |
|theaterPort               |18056                      |By default, 18056 is for game servers and        |
|                          |                           |18275 for game clients                           |

#### TXN = MemCheck, FESL server => game client/server
This message is sent every 10 seconds, and acts a heartbeat packet. 
If either party stops receiving the MemCheck messages, connection loss is assumed.
Maybe an anti-tampering measure?

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|memcheck.[]               |0                          |                               |
|salt                      |5                          |                               |

#### TXN = MemCheck, game client/server => FESL server
This message is always a response to a MemCheck query message by the FESL server.

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|result                    |*empty*                    |                               |


#### TXN = NuLogin, game client/server => FESL server
This message is sent by clients/servers to authenticate.

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|returnEncryptedInfo       |0                          |                               |
|nuid                      |XxX_b3stP1ayer_XxX         |                               |
|password                  |thisIsAPassword            |                               |
|macAddr                   |$31dc51d43797              |                               |

#### TXN = NuLogin, FESL server => game client/server, on error

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|localizedMessage          |Incorrect password.        |                               |
|errorContainer.[]         |0                          |                               |
|errorCode                 |122                        |                               |

#### TXN = NuLogin, FESL server => game client/server, on success

|Key                       |Example value              |Note                                    |
|--------------------------|---------------------------|----------------------------------------|
|profileId                 |1                          |                                        |
|userId                    |1                          |                                        |
|nuid                      |XxX_b3stP1ayer_XxX         |                                        |
|lkey                      |OwPcFq[xA338SppTjx0Ybw4c   |A 24 character BF2Random (see Appendix) |


#### TXN = NuGetPersonas, game client/server => FESL server
This message is a query to lookup all characters owned by a user.

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|                          |                           |                               |

#### TXN = NuGetPersonas, FESL server => game client/server

|Key                       |Example value              |Note                                             |
|--------------------------|---------------------------|-------------------------------------------------|
|personas.*i*              |xXx_1337Sn1per_xXx         |One entry for every character owned by the user. |
|                          |                           |Contains the character name.                     |
|                          |                           |*i* is the zero-based index of the character.    |
|personas.[]               |1                          |The total amount of characters.                  |


#### TXN = NuGetAccount, game client/server => FESL server
This message retrieves general account information, based on the parameters sent.

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|                          |                           |                               |

#### TXN = NuGetAccount, FESL server => game client/server


|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|heroName                  |xXx_1337Sn1per_xXx         |                               |
|nuid                      |email@account.com          |                               |
|DOBDay                    |1                          |Date Of Birth                  |
|DOBMonth                  |1                          |                               |
|DOBYear                   |2017                       |                               |
|userId                    |1                          |                               |
|globalOptin               |0                          |                               |
|thidPartyOptin            |0                          |                               |
|language                  |enUS                       |                               |
|country                   |US                         |                               |


#### TXN = NuLoginPersona, game client/server => FESL server
This message is sent to login to a character/server.

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|name                      |My-Awesome-Server          |                               |

#### TXN = NuLoginPersona, FESL server => game client/server

|Key                       |Example value              |Note                                    |
|--------------------------|---------------------------|----------------------------------------|
|lkey                      |OwPcFq[xA338SppTjx0Ybw4c   |A 24 character BF2Random (see Appendix) |
|profileId                 |1                          |                                        |
|userId                    |1                          |                                        |


#### TXN = GetStatsForOwners, game client/server => FESL server
This message is sent to retrieve info for the character selection screen.

|Key                       |Example value              |Note                                     |
|--------------------------|---------------------------|-----------------------------------------|
|keys.*i*                  |c_ltm                      |One entry for every stat to be retrieved |
|keys.[]                   |1                          |Amount of stats to be retrieved          |
|owner                     |2                          |                                         |
|ownerType                 |1                          |                                         |
|periodId                  |0                          |                                         |
|periodPast                |0                          |                                         |

#### TXN = GetStatsForOwners, FESL server => game client/server

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|stats.*i*.ownerId         |1                          |                               |
|stats.*i*.ownerType       |1                          |                               |
|stats.*i*.stats.*j*.key   |level                      |                               |
|stats.*i*.stats.*j*.value |3.0000                     |                               |
|stats.*i*.stats.*j*.text  |3.0000                     |                               |
|stats.*i*.stats.[]        |1                          |                               |
|stats.[]                  |1                          |                               |


#### TXN = GetStats, game client/server => FESL server
This message is sent to retrieve info about a character/user.

|Key                       |Example value              |Note                                     |
|--------------------------|---------------------------|-----------------------------------------|
|owner                     |35                         |                                         |
|ownerType                 |1                          |                                         |
|periodId                  |0                          |                                         |
|periodPast                |0                          |                                         |
|keys.*i*                  |c_items                    |One entry for every stat to be retrieved |
|keys.[]                   |1                          |Amount of stats to be retrieved          |

#### TXN = GetStats, FESL server => game client/server

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|ownerId                   |2                          |                               |
|ownerType                 |1                          |                               |
|stats.*i*.key             |edm                        |                               |
|stats.*i*.value           |*empty*                    |                               |
|stats.*i*.text            |*empty*                    |                               |
|stats.[]                  |2                          |                               |


#### TXN = NuLookupUserInfo, game client/server => FESL server
This message is sent to retrieve basic information about a user.

|Key                       |Example value              |Note                              |
|--------------------------|---------------------------|----------------------------------|
|userInfo.*i*.userName     |xXx_1337Sn1per_xXx         |Names of the characters to lookup |
|userInfo.[]               |1                          |Amount of characters to lookup    |

#### TXN = NuLookupUserInfo, FESL server => game client/server

|Key                       |Example value              |Note                              |
|--------------------------|---------------------------|----------------------------------|
|userInfo.*i*.userName     |xXx_1337Sn1per_xXx         |                                  |
|userInfo.*i*.userId       |1                          |                                  |
|userInfo.*i*.masterUserId |1                          |                                  |
|userInfo.*i*.namespace    |MAIN                       |                                  |
|userInfo.*i*.xuid         |24                         |                                  |
|userInfo.*i*.cid          |1                          |                                  |
|userInfo.[]               |3                          |Amount of users to lookup info of |


#### TXN = GetPingSites, game client/server => FESL server
This message is a query for a list of endpoints to test for the lowest latency on a game client.

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|                          |                           |                               |

#### TXN = GetPingSites, FESL server => game client/server

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|minPingSitesToPing        |2                          |                               |
|pingSites.*i*.addr        |45.77.66.233               |                               |
|pingSites.*i*.name        |gva                        |                               |
|pingSites.*i*.type        |0                          |                               |
|pingSites.[]              |4                          |                               |


#### TXN = UpdateStats, game client/server => FESL server
This message is sent to update character stats.

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|u.*i*.o                   |2                          |owner                          |
|u.*i*.ot                  |1                          |ownerType                      |
|u.*i*.s.*j*.k             |c_slm                      |key                            |
|u.*i*.s.*j*.ut            |0                          |updateType                     |
|u.*i*.s.*j*.t             |*empty*                    |text                           |
|u.*i*.s.*j*.v             |1.0000                     |value                          |
|u.*i*.s.*j*.pt            |0                          |                               |
|u.*i*.s.[]                |1                          |Amount of stats to query       |
|u.[]                      |1                          |Amount of character to query   |

#### TXN = UpdateStats, FESL server => game client/server

|Key                       |Example value              |Note                             |
|--------------------------|---------------------------|---------------------------------|
|u.*i*.o                   |                           |Values are copied from the query |
|u.*i*.s.*j*.k             |                           |                                 |
|u.*i*.s.*j*.ut            |                           |                                 |
|u.*i*.s.*j*.t             |                           |                                 |
|u.*i*.s.*j*.v             |                           |                                 |
|u.*i*.s.*j*.pt            |                           |                                 |
|u.*i*.s.[]                |                           |                                 |
|u.[]                      |                           |                                 |


#### TXN = GetTelemetryToken, game client/server => FESL server
Returns a unique token for game telemetry.

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|                          |                           |                               |

#### TXN = GetTelemetryToken, FESL server => game client/server

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|telemetryToken            |MTU5LjE1My4yMzUuMjYsOTk0Nix|                               |
|                          |lblVTLF7ZmajcnLfGpKSJk53K/4|                               |
|                          |WQj7LRw9asjLHvxLGhgoaMsrDE3|                               |
|                          |bGWhsyb4e6woYKGjJiw4MCBg4bM|                               |
|                          |srnKibuDppiWxYKditSp0amvhJm|                               |
|                          |StMiMlrHk4IGzhoyYsO7A4dLM26|                               |
|                          |rTgAo%3d                   |                               |
|enabled                   |US                         |                               |
|filters                   |*empty*                    |                               |
|disabled                  |*empty*                    |                               |


#### TXN = Start, game client/server => FESL server
This message is sent to initiate a "playnow".

|Key                               |Example value              |Note                                  |
|----------------------------------|---------------------------|--------------------------------------|
|partition.partition               |/eagames/bfwest-dedicated  |                                      |
|debugLevel                        |high                       |                                      |
|players.*i*.ownerId               |2                          |                                      |
|players.*i*.ownerType             |1                          |                                      |
|players.*i*.props.{*propertykey*} |3                          |Example *propertykey* is pref-lvl_avg |
|players.[]                        |1                          |                                      |

#### TXN = Start, FESL server => game client/server

|Key                       |Example value              |Note                           |
|--------------------------|---------------------------|-------------------------------|
|id.id                     |1                          |                               |
|id.partition              |/eagames/bfwest-dedicated  |                               |

## Magma server

On startup, the game will connect to an HTTPS server. Like the FESL server, this connection is encrypted using TLS. After patching, the same certificate can be used that is used on the FESL server. Multiple Magma server domains are baked in the game executable. A command line argument can be used to switch between these domains, however this option seems to be disabled.

|Version       |Product     |Offset     |
|--------------|------------|-----------|
|1.46.222034.0 |Game client |0x009DEAD8 |
|1.42.217478.0 |Game server |0x00694E50 |

The following HTTPS paths are defined in the game executable:
* /
* /dc/submit
* /nucleus/authToken
* /nucleus/check/%s/%I64d
* /nucleus/entitlement/%s/status/%s
* /nucleus/entitlement/%s/useCount/%d
* /nucleus/entitlements/%I64d
* /nucleus/entitlements/%I64d
* /nucleus/entitlements/%I64d?entitlementTag=%s
* /nucleus/name/%I64d
* /nucleus/personas/%s
* /nucleus/refundAbilities/%I64d
* /nucleus/wallets/%I64d
* /nucleus/wallets/%I64d/%s/%d/%s
* /ofb/products
* /ofb/purchase/%I64d/%s
* /persona
* /relationships/acknowledge/nucleus:%I64d/%I64d
* /relationships/acknowledge/server:%s/%I64d
* /relationships/decrease/nucleus:%I64d/nucleus:%I64d/%s
* /relationships/decrease/nucleus:%I64d/server:%s/%s
* /relationships/decrease/server:%s/nucleus:%I64d/%s
* /relationships/increase/nucleus:%I64d/nucleus:%I64d/%s
* /relationships/increase/nucleus:%I64d/server:%s/%s
* /relationships/increase/server:%s/nucleus:%I64d/%s
* /relationships/roster/nucleus:%I64d
* /relationships/roster/nucleus:%I64d
* /relationships/roster/server:%s
* /relationships/roster/server:%s/bvip/1,3
* /relationships/status/nucleus:%I64d
* /relationships/status/server:%s
* /user
* /user/updateUserProfile/%I64d

### /nucleus/authToken
  For game servers:
 
    ```<success><token>$serverKey$</token></success>```
 
  with `$serverKey$` equal to the value of the `X-SERVER-KEY` cookie.
 
  For game clients:
 
    ```<success><token code="NEW_TOKEN">$userKey$</token></success>```
 
  with `$userKey$` equal to the value of the `magma` cookie.

### /relationships/roster/{type}:{id}
    ```
    <update>
        <id>1</id>
        <name>Test</name>
        <state>ACTIVE</state>
        <type>server</type>
        <status>Online</status>
        <realid>$id$</realid>
    </update>
    ```
  with $id$ equal to the id in the URL.

### /nucleus/entitlements/{heroID}

    ```
    <?xml version="1.0" encoding="UTF-8" standalone="yes" ?>
    <entitlements>
        <entitlement>
            <entitlementId>1</entitlementId>
            <entitlementTag>WEST_Custom_Item_142</entitlementTag>
            <status>ACTIVE</status>
            <userId>$heroID$</userId>
        </entitlement>
        <entitlement>
            <entitlementId>1253</entitlementId>
            <entitlementTag>WEST_Custom_Item_142</entitlementTag>
            <status>ACTIVE</status>
            <userId>$heroID$</userId>
        </entitlement>
    </entitlements>
    ```
  with $heroID$ equal to heroID in the URL.

### /nucleus/wallets/{heroID}
    ```
    <?xml version="1.0" encoding="UTF-8" standalone="yes" ?>
    <billingAccounts>
        <walletAccount>
            <currency>hp</currency>
            <balance>1</balance>
        </walletAccount>
    </billingAccounts>
    ```

### /ofb/products
    ```
    <?xml version="1.0" encoding="UTF-8" standalone="yes" ?>
    <products>
        <product overwrite="false" default_locale="en_US" status="Published" productId="">
            <status>Published</status>
            <productId>142</productId>
            <productName>WEST_Custom_Item_142</productName>
            <attributes>
                <attribute value="WEST_Custom_Item_142" name="productName"/>
                <attribute value="WEST_Custom_Item_142" name="Product Name"/>
                <attribute value="WEST_Custom_Item_142" name="Long Description"/>
                <attribute value="WEST_Custom_Item_142" name="Short Description"/>
                <attribute value="BFHPC_Neck" name="groupName"/>
                <attribute value="WEST_Custom_Item_142" name="entitlementTag"/>
                <attribute value="142" name="entitlementId"/>
                <attribute value="1" name="sortKey"/>
                <attribute value="1" name="duration" />
                <attribute value="MONTH" name="durationType"/>
            </attributes>
        </product>
        <product overwrite="false" default_locale="en_US" status="Published" productId="">
            <status>Published</status>
            <productId>142</productId>
            <productName>WEST_Custom_Item_142</productName>
            <attributes>
                <attribute value="WEST_Custom_Item_141" name="productName"/>
                <attribute value="WEST_Custom_Item_141" name="Product Name"/>
                <attribute value="WEST_Custom_Item_141" name="Long Description"/>
                <attribute value="WEST_Custom_Item_141" name="Short Description"/>
                <attribute value="BFHPC_Neck" name="groupName"/>
                <attribute value="WEST_Custom_Item_141" name="entitlementTag"/>
                <attribute value="141" name="entitlementId"/>
                <attribute value="1" name="sortKey"/>
                <attribute value="1" name="duration" />
                <attribute value="MONTH" name="durationType"/>
            </attributes>
        </product>
        <product overwrite="false" default_locale="en_US" status="Published" productId="">
            <status>Published</status>
            <productId>142</productId>
            <productName>WEST_Custom_Item_142</productName>
            <attributes>
                <attribute value="WEST_Custom_Item_140" name="productName"/>
                <attribute value="WEST_Custom_Item_140" name="Product Name"/>
                <attribute value="WEST_Custom_Item_140" name="Long Description"/>
                <attribute value="WEST_Custom_Item_140" name="Short Description"/>
                <attribute value="BFHPC_Neck" name="groupName"/>
                <attribute value="WEST_Custom_Item_140" name="entitlementTag"/>
                <attribute value="140" name="entitlementId"/>
                <attribute value="1" name="sortKey"/>
                <attribute value="1" name="duration" />
                <attribute value="MONTH" name="durationType"/>
            </attributes>
        </product>
    </products>
    ```

## Theater

The third type of connection is the Theater connection which runs over both TCP and UDP. 
A seperate set of network sockets is made for the game servers and the game clients. 
Theater connections are mostly in plaintext.
The Theater network address and port is received by the game server/client through the FESL Hello message.

Packets received or sent from the UDP port are decoded/encoded using the "gamespy XOR". (See Appendix)


## Appendix

### FESL certificate patch

Source: http://aluigi.altervista.org/patches/fesl.lpatch

```
====================================================================================
#
# this file has been created for the Lame patcher program available for both *nix
# and Windows platforms.
# You need this program for continuing the patching of your files:
#
#   http://aluigi.org/mytoolz.htm#lpatch
#
# Quick step-by-step for Windows:
# - launch lpatch.exe
# - select this fesl.lpatch file
# - read the message windows and click yes
# - select the file (usually executables or dlls) to patch
# - read the message windows to know if everything has been patched correctly

TITLE
    EA games fesl.ea.com certificate verification remover 0.2
    by Luigi Auriemma
    e-mail: aluigi@autistici.org
    web:    aluigi.org

INTRO
    this modification removes the verification of the SSL certificate
    sent by the *.fesl.ea.com server (ports 18240,18020,18120,18081,
    18125,18270,18060,18210,18310 and others) when an EA game logins
    on it.
    this login mechanism is used with clients and servers so this
    remover is intended to both.
    .
    the result of such modification is that the game will not longer
    drop the connection if the fesl server certificate is invalid so
    the users can build their own "fesl" server for LAN gaming and
    other possible things like understanding the protocol (fsys, acct,
    TXN, ... very simple, use stcppipe -S for dumping the whole
    decrypted connection).
    .
    some games that use fesl are Battlefield 2142 / Heroes,
    Command & Conquer 3, The Lord of the Rings, Medal of Honor Airborne,
    NASCAR, Need for Speed Carbon / Undercover, Mercenaries 2, Dragon
    Age and many others.
    .
    note that the executable must be NOT encrypted or compressed (that
    happens when are used CD protections).

FILE
    *.exe

MAX_CHANGES
    1


BYTES_ORIGINAL
    81 ?? EE 0F 00 00   ; AND ECX,0FEE
    83 ?? 15            ; ADD ECX,15
    8B ??               ; MOV EAX,ECX

BYTES_PATCH
    ?? ?? ?? ?? ?? ??
    B8 15 00 00 00      ; MOV EAX, 15


BYTES_ORIGINAL
    B8 03 10 00 00      ; MOV EAX,1003
    5D                  ; POP EBP

BYTES_PATCH
    B8 15 00 00 00      ; MOV EAX, 15

====================================================================================
```

### Generating a BF2Random

A BF2Random of length `n` consists of `n` characters chosen randomly from the following string:
`0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ][`

### Performing a "gamespy XOR"

Given a string with n characters, the string is encoded as follows:
```
string encode(inputstring):
    m = length of "gameSpy"
	for i = 0 to n
		inputchar = inputstring[i]
		xorChar = "gameSpy"[i mod m]
		outputchar[i] = inputchar XOR xorChar
```

### Game server arguments

Obtained using `BFHeroes_w32ded.exe +?`. Some of these are remnants of Battlefield 2/2142 and may not be functional.

```
Usage: BFHeroes.exe <options> 

Available options are: 
+dedicated - Start in dedicated server mode 
+multi - Allow starting multiple BF2 instances 
+joinServer - Join a server by ip address or hostname 
+playerName - Set the player name 
+password - Set the server password when joining a server 
+config - Sets path to the ServerSettings.con file to use 
+mapList - Sets the path to the MapList.con file to use 
+lowPriority - Run the game with slightly lower priority 
+loadLevel - Set the level to load 
+fullscreen - Start game in full screen mode 
+noSound - Start game without sound 
+demo - Sets the con-file with demo options 
+maxPlayers - Sets max players. 
+gameMode - Sets the game mode. 
+modPath - Set the mod path (default mods/bfheroes) 
+showAsserts - Show non-content asserts 
+showContentAsserts - Show content asserts 
+help - Displays this help 
+? - Same as +help 
+ranked - Allows stats submission to backend servers 
+overlayPath - Start game with a custom path for configuration files 
+port - specifies the network port to be used 
+pbPath - Set the path to use for PunkBuster on multi-instance configurations (defaults to (install_dir)/pb 
+eaEncyptedLogin - Encyrpted login used to connect to the EA backend 
+soldierName - Auto-login to a soldier in the specified EA Account Name 
+sessionld - Magma session identifier 
+dataCenters - prioritized list of data centers, showing the order in which the client should connect to data centers
+webBrowser - Enable or disable the embedded web browser component 
+magmaEnvironment - Sets the magma environment to use 
+plasmaEnvironment - Sets the plasma environment to use 
+lang - Sets the language to use 
+dc - Toggles data collection 
+eaAccountName - Auto-login with the specified EA Account Name 
+eaAccountPassword - Password to the specified EA Account Name 
+key - Server API Key for Magma 
+guid - Server GUID 
+secret - Secret which is associated with the server GUID 
+useServerMonitorTool - Use the server monitor tool 
+serverMonitorAddress - Server monitor tool address 
+serverMonitorPort - Server monitor tool port 
+checkEntitlement - Perform an entitlement check at login/connection (used for beta) 
+magma - Use Magma backend 
+magmaProtocol - Select HTTP or HTTPS protocol for hydra/Magma requests 
+plasmaFindListOfServers - Search for a list of servers (as opposed to just one). 
+plasmaServerName - Specify a server name to filter for during playnow 
+plasmaClientPort - Specify a port for the plasma client to use for game data 
+autoLogin - Automatically login at application start, before reaching the frontend 
+survey - Launches survey at regular intervals after game shuts down. 
+webSiteHostName - Specify which website the game is launched from. 
+battleFundsHostName - Specify which website to buy battlefunds from.

Advanced options: 
+hostServer 
+ai 
+provider 
+region 
+type 
```
