protocol_blueprint.md

Framing Rules (applies to all message types): 
 Every JSON object is UTF-8 encoded and terminated by a newline character \n (0x0A). The receiver accumulates incoming bytes into a stream buffer until a \n is encountered, extracts the complete line, and deserializes the JSON object. Leftover after the last /n stay in the buffer until the rest of the message arrives. JSON MUST be a single line. 


### CONNECT

- **Direction:** Client -> Server
- **Purpose:** Request to join the game room. First message sent after
  the TCP connection opens.

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
| `player_id` | string  | yes      | Client's name, 1-16 chars (`A-Z a-z 0-9 _`)       |
| `payload`   | object  | yes      | Contains the fields in the payload table below    |
| `timestamp` | integer | yes      | Unix epoch seconds when the message was created   |

#### Payload fields:

| Payload field | Type   | Required | Description                                                         |
|---------------|--------|----------|---------------------------------------------------------------------|
| `players_connected` | integer | yes      | Must be at least one (the one waiting for the other player)  |
| `Notification` | string | yes      | 1-16 chars (`A-Z a-z 0-9 _`) Tells server is waiting for other player    |





GAME_START:

MOVE:

STATE_UPDATE:

ERROR:

DISCONNECT:

GAME_OVER:




Concrete Framing Rule & Wire Examples: Concrete examples of your chosen framing mechanism on the continuous wire stream (e.g., JSON schema with newline delimiters
