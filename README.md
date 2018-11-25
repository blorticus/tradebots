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

The Player API relies on a text-based protocol.  A client sends a command followed by zero or more attributes, then a newline (ASCII 10 -- a \n).
