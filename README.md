## Overview

This is a space trading game with an access API.  The game has a Map that
consists of Sectors.  Sectors are interconnected essentially randomly.
Players start with a Capital Ship, which has Holds.  Holds can contain
Trade Goods.  Players can buy and sell Trade Goods at Trade Ports.  They
can also produce Trade Goods on Planets, which are spread throughout the
Map.  Trades yield Credits.

Players can also purchase Fighters, which when put together, are Fleets.
Fighters can be produced at Planets by consuming Trade Goods.  They can
also be purchased at Neutral Zone Planets.

## Starting the Application

The application is started as follows:

```
tradebot <config_file_yaml>
```

It must be provided a YAML file as follows:

```
---
Transport:
   # default value is ['::', '0.0.0.0']
   Addresses: [<ip1>, ...]

   # default value is 33666
   Port: <bind_port>

   # not required.  If omitted, TCP only
   TLS:
      Certificate: /path/to/cert
      PrivateKey: /path/to/key

   # default value is 10 concurrent connections
   ConnectionLimit: <n>

# where to store data for game
DataRepository:
   File: /path/to/repository

# Default sector count is 2500
Map:
   Sectors: <n>

# Administrative user name and password for access
# to administrative API
Admin:
   Name: <name>
   Password: <passwd>
```

## Administrative API


## Player API

The Player API relies on a text-based protocol.  All encodings are UTF-8.  A client sends a command followed by zero or more attributes, then a newline (ASCII 10 -- a \n).  The server responds with a response code three digit number, a text description of the response code (not to exceed 256 UTF-8 characters) a content length of the body in octets (encoded integer as text digits, with a value that must be less than or equal to 65535; no negative numbers), a newline, then the body, which must be empty (content length is zero) or a valid JSON encoding.  The server may also send unsolicited messages encoded like response messages (in this case, they are called "notifications"), so the client should asynchronously listen for server messages.

In general, any client command can generate one of the following responses (any text preceded by a dollar has a variable value; all else are literals, and are case-sensitive):

* `300 USER_NOT_LOGGED_IN 0\n` -- when the user has not yet logged in successfully.  The server will immediately disconnect the transport.
* `400 COMMAND_NOT_UNDERSTOOD 0\n` -- when the supplied command is not recognized.
* `401 INVALID_COMMAND_PARAMETERS $len\n{ parameter_name: "$name" }` -- when one or more parameters are not recognized.  Only the first invalid parameter will be identified.
* `402 INVALID_PARAMETER_VALUE $len\n{ parameter_name: "$name", message: "$message" }` -- when one or more parameter have an invalidly formatted value.  Only the first invalid parameter will be identified.  `message` is an error message, describing in human-readable text why the valid is invalid.
* `500 TRANSIENT_SERVER_FAILUrE $len\n{ message: "$message" }` -- when a general server failure has occurred but is probably recoverable.  The user remains logged in.
* `501 NON_TRANSIENT_SERVER_FAILURE $len\n{ message: "$message" }` -- when a general server failure has occurred and it is not recoverable.  The user will no longer be logged in, and the server may optionally close the transport connection.
* `502 I_DONT_LIKE_YOU 0\n` -- when, for whatever reason, the server will not accept or will no longer support the transport.  The transport will be closed, and additional connection attempts from the client IP may be forbidden, at least temporarily.  This is a general protection mechanism (e.g., if the client starts sending garbage input).

The client commands and valid responses (in addition to the general responses identified above) are as follows (any text preceded by a dollar has a variable value; all else are literals, and are case-sensitive):

* `LOGIN NAME $ulen $user PASSWORD $plen $password` -- Attempt to log the user in.  This must be the first action in a transport session.  Any other action will result in a `300` response and transport disconnect.  `$ulen` is the octet length of the `$user` (which is a UTF-8 text stream that must not exceed 32 code points).  `$plen` is the octet length of the `$password` (which is a UTF-8 text stream that must not exceed 32 code points).
  - `200 SUCCESS 0\n` -- User is now logged in, and the session is considered authenticated.
  - `301 LOGIN_FAILURE 0\n` -- Attempt to log in with a user name that isn't recognized, an account that is locked out, or an incorrect password is supplied for the user.  Immediately terminate transport on this failure.

* `LOGOUT` -- Attempt to terminate the current session.  This is universally successful.
  - `200 SUCCESS 0\n` -- Transport is terminated.
  
* `LOOK` -- Retrieve a detailed description of the current sector in which the player's capital ship is located.
  - `200 SUCCESS $len\n{ $sector_details }` -- Provide the sector details.  See the section **REPSONSE BODY DETAILS** below.
  
* `LOOK AT $sector` -- Attempt to retrieve details of a sector by number.  `$sector` must be a text-encoded set of digits.
  - `200 SUCCESS $len\n{ $sector_details }`
  - `403 DISALLOWED 0\n` -- If the player has nothing in the sector that would provide those details.
  - `404 NO_SUCH_SECTOR 0\n` -- If there is no such sector.

* `MOVE TO $sector` -- Attempt to move the capital ship to a sector by number.
  - `200 SUCCESS $len\n{ $sector_details }` -- Provides the details of the sector into which the ship has moved.
  - `405 CANNOT_GET_THERE 0\n` -- If `$sector` is not adjascent to the current sector.
  
* `MOVE UNIT $num TO $sector` -- Attempt to move a non-capital ship object to the identified sector.  `$num` is a valid unit id, which is text-encoded digits.
  - `200 SUCCESS $len\n{ $sector_details }` -- Provides the details of the sector into which the unit has moved.
  - `405 CANNOT_GET_THERE 0\n` -- If `$sector` is not adjascent to the current sector for the unit.
  - `406 NO_SUCH_UNIT 0\n` -- If `$unit` does not identify an actual unit owned by the current player.

* `MOVE UNIT $num ALONG $path` -- Some units can move an arbitrary distance in a single game turn.  However, the player must supply the complete path, where `$path` is `$sector,$sector,$sector,...`.
  - `200 SUCCESS $len\n{ $sector_details_list }` -- Provides the details of each sector through which the unit has moved.
  - `405 CANNOT_GET_THERE $len\n{ $sector_details_list }` -- If one of the supplied sectors in the list is not properly connected to the previously sector list member.  `$sector_details_list` will contain information about each sector through which the unit successfully moved (and thus, the client should be able to infer where the failure occurred).  This will also occur if the unit cannot move the total number of spaces in the list.
  - `406 NO_SUCH_UNIT 0\n` -- If `$unit` does not identify an actual unit owned by the current player.

Notifications from the server include:

* `100 SECTOR_UPDATE $len\n{ $sector_details }` -- When something changes in a sector occupied by a player unit.
* `502 ...` as above.
* `101 SHUTTING_DOWN $len\n{ mesasge: "$message" }` -- When the server is about to shut down.  A human-readable reason is usually provided in `$message`.  The transport will immediately disconnect, and no further client messages will be accepted (and may result in a reset).

