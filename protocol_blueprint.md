protocol_blueprint.md

Framing Rules (applies to all message types): 
 Every JSON object is UTF-8 encoded and terminated by a newline character \n (0x0A). The receiver accumulates incoming bytes into a stream buffer until a \n is encountered, extracts the complete line, and deserializes the JSON object. Leftover after the last /n stay in the buffer until the rest of the message arrives. JSON MUST be a single line. 


### CONNECT

- **Direction:** Client -> Server
- **Purpose:** Request to join the game room. First message sent after the TCP connection opens.

#### Envelope fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `msg_type`  | string  | yes      | Always `"CONNECT"`                                |
| `player_id` | string  | yes      | Client's name, 1-16 chars (`A-Z a-z 0-9 _`)       |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Payload field | Type   | Required | Description                                                         |
|---------------|--------|----------|---------------------------------------------------------------------|
| `name`        | string | yes      | 1-16 chars (`A-Z a-z 0-9 _`). Must equal to `player_id`             |

#### Sample message:

```json
{"msg_type":"CONNECT","player_id":"Alice","payload":{"name":"Alice"},"timestamp":1727000000}
```

#### Wire stream example:

```text
{"msg_type":"CONNECT","player_id":"Alice","payload":{"name":"Alice"},"timestamp":1727000000}\n{"msg_type":"DISCONNECT","player_id":"Alice","payload":{},"timestamp":1727000001}\n
```



### LOBBY_WAIT:

- **Direction:** Server -> Client
- **Purpose:** Notification that the server is waiting for Player 2

#### Envelope fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `msg_type`  | string  | yes      | Always `"LOBBY_WAIT"`                             |
| `player_id` | string  | yes      | Always `"SERVER"` for server messages             |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Payload field | Type   | Required | Description                                                         |
|---------------|--------|----------|---------------------------------------------------------------------|
| `players_connected` | integer | yes      | Always one (the one waiting for the other player)            |
| `notification` | string | yes      | 1-50 chars. Tells player that server is waiting for other player    |

#### Sample message:

```json
{"msg_type":"LOBBY_WAIT","player_id":"SERVER","payload":{"players_connected":1,"notification":"Waiting for an opponent to appear..."},"timestamp":1727000001}
```

#### Wire stream example:

```text
{"msg_type":"LOBBY_WAIT","player_id":"SERVER","payload":{"players_connected":1,"notification":"Waiting for an opponent to appear..."},"timestamp":1727000001}\n{"msg_type":"GAME_START","player_id":"SERVER","payload":{"role":"Player_1","opponent_name":"Bob","grid_size":10,"ship_lengths":[5,4,3,2]},"timestamp":1727000030}\n
```


### GAME_START:

- **Direction:** Server -> Clients
- **Purpose:** Game initiated, assigns roles (e.g. Player 1 vs Player 2). Explains the game parameters.

#### Envelope fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `msg_type`  | string  | yes      | Always `"GAME_START"`                             |
| `player_id` | string  | yes      | Always `"SERVER"` for server messages             |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Payload field | Type   | Required | Description                                                         |
|---------------|--------|----------|---------------------------------------------------------------------|
| `role` | string | yes      | Player_1 or Player_2, chosen randomly by server                            |
| `opponent_name` | string | yes      | Client's name, 1-16 chars (`A-Z a-z 0-9 _`)                       |
| `grid_size` | integer | yes      | 20 (20x20)                                                           |
| `ships_size` | array | yes      | 6 ships sizes [5,9,6,2,7,5] all are placed vertically (for simplicity)|

#### Sample message:

```json
{"msg_type":"GAME_START","player_id":"SERVER","payload":{"role":"Player_1","opponent_name":"Evil_Alice", "grid_size":20, "ships_size":[5,9,6,2,7,5]},"timestamp":1727000001}
```

#### Wire stream example:

```text
{"msg_type":"GAME_START","player_id":"SERVER","payload":{"role":"Player_1","opponent_name":"Evil_Alice","grid_size":20,"ships_size":[5,9,6,2,7,5]},"timestamp":1727000001}\n
```



### PLACE_SHIPS:

- **Direction:** Clients -> server
- **Purpose:** Players place their ships on the grid and tell server that they are ready to start the game. There is a rule, no two ships may share a cell.

#### Envelope fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `msg_type`  | string  | yes      | Always `"PLACE_SHIPS"`                            |
| `player_id` | string  | yes      | Client's name, 1-16 chars (`A-Z a-z 0-9 _`)       |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `ships`     | array (ships)| yes | 6 entries.                                        |

Each ship object:

| Field    | Type    | Required | Description                                                                 |
|----------|---------|----------|-----------------------------------------------------------------------------|
| `length` | integer | yes      | Ship length, one of the values in `ships_size` array                        |
| `row`    | integer | yes      | 0-19, row of the ship's top cell. The ship extends downward, so `row + length <= 20` |
| `col`    | integer | yes      | 0-19, column the ship occupies (all ships are vertical)                     |

#### Sample message:

```json
{"msg_type":"PLACE_SHIPS","player_id":"Alice","payload":{"ships":[{"length":5,"row":0,"col":0},{"length":9,"row":0,"col":2},{"length":6,"row":0,"col":4},{"length":2,"row":0,"col":6},{"length":7,"row":0,"col":8},{"length":5,"row":10,"col":0}]},"timestamp":1727000008}
```

#### Wire stream example:

```text
{"msg_type":"PLACE_SHIPS","player_id":"Alice","payload":{"ships":[{"length":5,"row":0,"col":0},{"length":9,"row":0,"col":2},{"length":6,"row":0,"col":4},{"length":2,"row":0,"col":6},{"length":7,"row":0,"col":8},{"length":5,"row":10,"col":0}]},"timestamp":1727000008}\n
```



### VERIFY_PLACEMENT:

- **Direction:** Server -> Client
- **Purpose:** Server verifies that ships were placed correctly. There are no ships sharing the same cell. 

#### Envelope fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `msg_type`  | string  | yes      | Always `"VERIFY_PLACEMENT"`                             |
| `player_id` | string  | yes      | Always `"SERVER"` for server messages             |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Payload field | Type   | Required | Description                                                         |
|---------------|--------|----------|---------------------------------------------------------------------|
| `ships_placed` | integer | yes    | Must be 6                                                         |
| `waiting_for_player` | boolean | yes | `true` if the opponent has not yet placed their ships, `false` if both are placed|

#### Sample message:

```json
{"msg_type":"VERIFY_PLACEMENT","player_id":"SERVER","payload":{"ships_placed":"6","waiting_for_player":false},"timestamp":1727000001}
```

#### Wire stream example:

```text
{"msg_type":"VERIFY_PLACEMENT","player_id":"SERVER","payload":{"ships_placed":6,"waiting_for_player":false},"timestamp":1727000010}\n{"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"turn_number":0,"last_move":null,"ships_remaining":{"Player_1":6,"Player_2":6},"active_player":"Player_1","final_turn":false},"timestamp":1727000011}\n
```

### STATE_UPDATE:

- **Direction:** Server -> Clients
- **Purpose:** Broadcast current game board / state and active player turn

#### Envelope fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `msg_type`  | string  | yes      | Always `"STATE_UPDATE"`                             |
| `player_id` | string  | yes      | Always `"SERVER"` for server messages             |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Payload field | Type   | Required | Description                                                         |
|---------------|--------|----------|---------------------------------------------------------------------|
| `turn_number` | integer | yes    | Must be 6                                                            |
| `last_move` | object | yes | null at the opening update                                                    |
| `ships_remaning_Player_1` | integer | yes | Number of ships remaining for player_1                      |
| `ships_remaning_Player_2` | integer | yes | Number of ships remaining for player_2                                                   |
| `active_player` | string | yes | Name of the player who must move next                                  |
| `final_turn` | boolean | yes | true when Player_1 has sunk all of Player_2's ships. Player_2 has one last chance|

last_move object:

| Field              | Type            | Required | Description                                                      |
|--------------------|-----------------|----------|------------------------------------------------------------------|
| `by`               | string          | yes      | Player_1 or Player_2, the player who fired                       |
| `row`              | integer         | yes      | 0-19                                                             |
| `col`              | integer         | yes      | 0-19                                                             |
| `result`           | string          | yes      | Hit! or miss message                                             |
| `sunk_ship_length` | integer or null | yes      | Length of the ship this shot sank, or   null if no ship was sunk |



#### Sample message:

```json
{"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"turn_number":4,"last_move":{"by":"Player_1","row":1,"col":2,"result":"hit!","sunk_ship_length":9},"ships_remaining_Player_1":4,"ships_remaining_Player_2":5,"active_player":"Player_2","final_turn":false},"timestamp":1727000500}
```

#### Wire stream example:

```text
{"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"turn_number":4,"last_move":{"by":"Player_1","row":1,"col":2,"result":"hit","sunk_ship_length":9},"ships_remaining_Player_1":4,"ships_remaining_Player_2":5,"active_player":"Player_2","final_turn":false},"timestamp":1727000500}\n{"msg_type":"STATE_UPDATE","player_id":"SERVER","payload":{"turn_number":6,"last_move":{"by":"Player_2","row":7,"col":3,"result":"miss","sunk_ship_length":null},"ships_remaining_Player_1":4,"ships_remaining_Player_2":5,"active_player":"Player_1","final_turn":false},"timestamp":1727000506}\n
```



### MOVE:

- **Direction:** Client -> Server
- **Purpose:** Player action (e.g., cell coordinates or answer choice)

#### Envelope fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `msg_type`  | string  | yes      | Always `"MOVE"`                                   |
| `player_id` | string  | yes      | Client's name, 1-16 chars (`A-Z a-z 0-9 _`)       |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Payload field | Type   | Required | Description                                                         |
|---------------|--------|----------|---------------------------------------------------------------------|
| `row` | integer | yes    | 0-19                                                                         |
| `col` | integer | yes | 0-19                                                                            |

#### Sample message:

```json
{"msg_type":"MOVE","player_id":"Alice","payload":{"row":6,"col":3},"timestamp":1727000001}
```

#### Wire stream example:

```text
{"msg_type":"MOVE","player_id":"Alice","payload":{"row":6,"col":3},"timestamp":1727000010}\n
```


### ERROR:

- **Direction:** Server -> Client
- **Purpose:** Invalid move or malformed packet error

#### Envelope fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `msg_type`  | string  | yes      | Always `"ERROR"`                                  |
| `player_id` | string  | yes      | Always `"SERVER"` for server messages             |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Payload field | Type   | Required | Description                                                         |
|---------------|--------|----------|---------------------------------------------------------------------|
| `code` | string | yes    | One of the code error (still figuring what errors will appear)                                                          |
| `detail` | string | yes | Explains what the code means                                                  |

#### Sample message:

```json
{"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"INVALID_PLACEMENT","detail":"Two ships share the same cell!"},"timestamp":1727000001}
```

#### Wire stream example:

```text
{"msg_type":"ERROR","player_id":"SERVER","payload":{"code":"INVALID_PLACEMENT","detail":"Two ships share the same cell!"},"timestamp":1727000001}\n
```



### DISCONNECT:

- **Direction:** Client -> Server
- **Purpose:** A player disconnects and connection is shut down gracefully

#### Envelope fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `msg_type`  | string  | yes      | Always `"DISCONNECT"`                             |
| `player_id` | string  | yes      | Client's name, 1-16 chars (`A-Z a-z 0-9 _`)       |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Payload field | Type   | Required | Description                                                         |
|---------------|--------|----------|---------------------------------------------------------------------|
| `reason`      | string | yes      | Always QUIT (the player is leaving on purpose)       |

#### Sample message:

```json
{"msg_type":"DISCONNECT","player_id":"Alice","payload":{"reason":"QUIT"},"timestamp":1727000001}
```

#### Wire stream example:

```text
{"msg_type":"DISCONNECT","player_id":"Alice","payload":{"reason":"QUIT"},"timestamp":1727000001}\n
```



### GAME_OVER:

- **Direction:** Server -> Clients
- **Purpose:** Victory / Draw notification with final scores

#### Envelope fields:

| Field       | Type    | Required | Description                                       |
|-------------|---------|----------|---------------------------------------------------|
| `msg_type`  | string  | yes      | Always `"GAME_OVER"`                              |
| `player_id` | string  | yes      | Always `"SERVER"` for server messages             |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Payload field | Type   | Required | Description                                                         |
|---------------|--------|----------|---------------------------------------------------------------------|
| `outcome`                  | string         | yes      | WIN, DRAW, or FORFEIT                          |
| `winner`                   | string or null | yes      | Player_1 or Player_2. `null` if the outcome is DRAW           |
| `loser`                    | string or null | yes      | Player_1 or Player_2. `null` if the outcome is DRAW. In a forfeit, the player who left |
| `reason`                   | string         | yes      | ALL_SHIPS_SUNK, OPPONENT_QUIT, or OPPONENT_DISCONNECTED (connection dropped) |
| `ships_remaining_Player_1` | integer        | yes      | Final count of Player_1's ships still afloat, 0-6                           |
| `ships_remaining_Player_2` | integer        | yes      | Final count of Player_2's ships still afloat, 0-6                           |
| `total_turns`              | integer        | yes      | Total number of accepted moves in the game                                  |


#### Sample message:

```json
{"msg_type":"GAME_OVER","player_id":"SERVER","payload":{"outcome":"WIN","winner":"Player_1","loser":"Player_2","reason":"ALL_SHIPS_SUNK","ships_remaining_Player_1":3,"ships_remaining_Player_2":0,"total_turns":90},"timestamp":1727000900}
```

#### Wire stream example:

```text
{"msg_type":"GAME_OVER","player_id":"SERVER","payload":{"outcome":"WIN","winner":"Player_1","loser":"Player_2","reason":"ALL_SHIPS_SUNK","ships_remaining_Player_1":3,"ships_remaining_Player_2":0,"total_turns":90},"timestamp":1727000900}\n\n
```

### Connection Termination and Socket Lifecycle Management 

#### 1. Graceful termination:
Active client sends a structured DISCONNECT message before terminating. Then calls socket.close(). 

The operating system then initiates the TCP 4-way FIN handshake. 

Server reads the DISCONNECT message, closes its side of the socket, and releases that player's resources.

If a game is in progress, it sets the state to GAME_OVER with outcome FORFEIT, then moves to CLEANUP. 

#### 2. Graceful TERMINATION without DISCONNECT
Client calls close() or exits normally without sending DISCONNECT. The operating system still sends a TCP FIN, and the server's next recv() on that socket returns 0 bytes (b"") in Python. 

This prevents an infinite loop, since without it the application would consume 100% CPU. 


#### 3. Abrupt termination 
If the network links fail or clients crash, low-level socket operations raise exceptions. 

-ConnectionResetError: Happens at recv() or sendall(). The server will handle it as DISCONNECT
-BrokenPipeError: Happens at sendall() to a close peer. The server will handle it as DISCONNECT
-TimeoutError: Happens at recv() when a socket timeout is set. The server will handle it as DISCONNECT

#### 4. Server behavior by state 
LOBBY_WAIT: close socket, remove player, stay in WAITING_FOR_PLAYERS. 
GAME_OVER OR CLEANUP: Close both sockets and finish the reset. 

IF both clients are gone, the server catches the exception and continues cleanup. 

If a line is not valid UTF-8 or valid JSON, or the buffer grows past 4096 bytes without containing a \n, the message is malformed. If there is no \n and it exceeds the limit, the stream cannot be resynchronized, so the server will close the connection. 
