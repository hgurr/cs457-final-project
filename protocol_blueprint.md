# Application Protocol Blueprint

## Message Types & Structured Schema Definitions
### 1. `CONNECT`
**Direction:** Client &rarr; Server

**Purpose:** Client requests to join the game with player alias.

**JSON Schema**
| Keys        | Data Type |
|:-----------:|:---------:|
| `message`   | string    |
| `player_id` | string    |

**Sample Payload**
```json
{
  "message": "CONNECT",
  "player_id": "Hallie"
}
```

### 2. `LOBBY_WAIT`
**Direction:** Server &rarr; Client

**Purpose:** Server notifies Client 1 that it is waiting for Client 2 to connect.

**JSON Schema**
| Keys        | Data Type |
|:-----------:|:---------:|
| `message`   | string    |
| `player_id` | string    |

**Sample Payload**
```json
{
  "message": "LOBBY_WAIT",
  "player_id": "Hallie"
}
```

### 3. `GAME_START`
**Direction:** Server &rarr; Client

**Purpose:** Server notifies both clients that the game has started and assigns the client a role (Player 1/Player 2).

**JSON Schema**
| Keys              | Data Type |
|:-----------------:|:---------:|
| `message`         | string    |
| `player_id`       | string    |
| `role`            | string    |
| `opponent_id`     | string    |
| `starting_player` | string    |

**Sample Payload**
```json
{
  "message": "GAME_START",
  "player_id": "Hallie",
  "role": "PLAYER_1",
  "opponent_id": "Allie",
  "starting_player": "PLAYER_1"
}
```

### 4. `MOVE`
**Direction:** Client &rarr; Server

**Purpose:** Active client submits a move to place a disc in a column.

**JSON Schema**
| Keys        | Data Type |
|:-----------:|:---------:|
| `message`   | string    |
| `player_id` | string    |
| `column`    | integer   |

**Sample Payload**
```json
{
  "message": "MOVE",
  "player_id": "Hallie",
  "column": 4
}
```

### 5. `STATE_UPDATE`
**Direction:** Server &rarr; Client

**Purpose:** Server broadcasts the updated board state and active client turn.

**JSON Schema**
| Keys            | Data Type   |
|:---------------:|:-----------:|
| `message`       | string      |
| `board`         | integer[][] |
| `active_player` | string      |

**Sample Payload**
```json
{
  "message": "STATE_UPDATE",
  "board": [
    [0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0],
    [0,0,0,0,0,0,0],
    [0,0,0,1,0,0,0]
  ],
  "active_player": "PLAYER_2"
}
```
`0` = empty
`1` = Player 1 
`2` = Player 2

### 6. `ERROR`
**Direction:** Server &rarr; Client

**Purpose:** Server notifies the client of an out-of-turn move or invalid coordinates.

**JSON Schema**
| Keys      | Data Type |
|:---------:|:---------:|
| `message` | string    |
| `error`   | string    |

**Sample Payload**
```json
{
  "message": "ERROR",
  "error": "OUT_OF_TURN"
}
```

### 7. `DISCONNECT`
**Direction:** Client &rarr; Server

**Purpose:** Client notifies the server of intentional departure/quit.

**JSON Schema**
| Keys        | Data Type |
|:-----------:|:---------:|
| `message`   | string    |
| `player_id` | string    |

**Sample Payload**
```json
{
  "message": "DISCONNECT",
  "player_id": "Hallie"
}
```

### 8. `GAME_OVER`
**Direction:** Server &rarr; Client

**Purpose:** Server broadcasts final game outcome (Winner/Draw/Forfeit).

**JSON Schema**
| Keys      | Data Type   |
|:---------:|:-----------:|
| `message` | string      |
| `result`  | string      |
| `winner`  | string/null |

**Sample Payload**

***Win***
```json
{
  "message": "GAME_OVER",
  "result": "WIN",
  "winner": "Hallie"
}
```
***Draw***
```json
{
  "message": "GAME_OVER",
  "result": "DRAW",
  "winner": null
}
```
***Forfeit***
```json
{
  "message": "GAME_OVER",
  "result": "FORFEIT",
  "winner": "Allie"
}
```

## TCP Stream Packet Framing & Boundary Handling
- **Transport Protocol:** TCP
- **Serialization Format:** JSON
- **Framing Mechanism:** Newline-delimited JSON (`\n`)
- **Framing Rule:** Each JSON message is encoded as UTF-8 and terminated by a newline character `\n`. The receiver reads the incoming TCP byte stream and uses the newline character to identify the end of each message. The receiver accumulates incoming bytes into a stream buffer until a `\n` is encountered. Once a `\n` is encountered, the receiver extracts the complete message and deserializes the JSON object.

	Every message sent on the connection follows the format: `JSON message` + `\n`

*Example:*
```json
{"message":"CONNECT","player_id":"Hallie"}\n
```

### Raw On-the-Wire Byte Stream Examples
TCP is a continuous byte-stream protocol without built-in message boundaries. A message may arrive in fragmentation or coalescing.

- **Fragmentation:** A message arrives divided between multiple `recv()` chunks.

   *Example:*
```json
recv() #1:
{"message":"MOVE","player_id":"Hall
```
```
recv() #2:
ie","column":4}\n
```
The receiver appends both bytes to its buffer and waits for the `\n` to be received before processing the complete message.

- **Coalescing:** Multiple messages arrive in a single `recv()` chunk.

  *Example:*
```json
{"message":"CONNECT","player_id":"Hallie"}\n{"message":"MOVE","player_id":"Hallie","column":4}\n
```
The receiver separates the messages using the `\n` delimiter.

### Receiver Extraction Logic
1. Receive incoming bytes from the TCP connection and append them to the stream buffer.
2. Search the buffer for a newline character (`\n`).
3. If a newline character is found, extract all bytes before the newline as one complete message.
4. Decode the extracted message from UTF-8 encoding and deserialize the decoded JSON into a message object.
5. Remove the processed message and its corresponding newline character from the buffer. 
6. Repeat steps 1-5 when another complete message is available in the buffer.
7. If no newline is present, wait for additional TCP bytes before attempting to process the message.

## Connection Termination & Socket Lifecycle Management
### Graceful Application Disconnection
When a client intentionally leaves the game, it sends a DISCONNECT message to the server.
```json
{"message":"DISCONNECT","player_id":"Hallie"}
```
The server handles the player's departure, performs cleanup, and using TCP FIN, closes the TCP connection.
### Abrupt Termination Handling
When an unexpected disconnect is detected, the server handles unexpected connection loss through:
- **TCP RST:** The server handles `ConnectionResetError` when a client connection is unexpectedly reset and treats the player as disconnected.
- **Network drop:** The server detects a socket error or timeout and treats the player as disconnected.
- **TCP 0-byte EOF:** If `recv()` returns 0 bytes, the server treats the connection as closed and exits the socket receive loop.
- **Broken pipe:** If the server attempts to send data to a client that has already disconnected, it catches `BrokenPipeError` and treats the player as disconnected.

The server handles the unexpected disconnect, performs cleanup, and handles the departure as a forfeit if the game is active.
