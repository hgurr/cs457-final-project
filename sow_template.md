# CS 457 Project Statement of Work (SOW) & Protocol Specification Template

**Student Name:** Hallie Gurr 
**Date:** 2026-09-22
**Course:** CS 457 - Computer Networks and the Internet
**Target Server Domain:** `server.gurr.edu`  

---

## 1. Game Selection & Scope (Sprint 0)

> Planning is going to be an iterative process through the sprints so you don't have to have all the details now. Focus on big overview concepts. You will be updating the SOW as we plan.
> You have a lot of freedom to choose a game. There are a couple caveats.  

> - It must run in the console. The lab nodes won't be able to handle extensive graphics.
> - It has to be self-contained. You can use a internet-connector to download you code, but because the architecture must run 5 nodes you won't be able to run 
> - You are encouraged to use python, but I'm not going to make it a strict requirement. The instructor and TA's ability to help with C or Rust, etc will be diminished in other languages.

### 1.1 Game Overview
- **Chosen Game:** Connect Four
- **Player Capacity:** 2 Players (Simulated via 2 CML Client nodes)
- **Game Summary:** Connect Four is a two-player, turn-based strategy game played on a board that consists of 7 columns and 6 rows. Players take turns dropping their game pieces into a column, and the piece falls to the lowest available position within that column. The objective of the game is to be the first player to connect four of their pieces in a horizontal, vertical, or diagonal line.

### 1.2 Core Game Rules & Win/Draw Conditions
- **Turn Mechanics:** When two players connect to the game server, the server assigns Player 1 to the first player to connect and Player 2 to the second. Player 1 takes the first turn. Players alternate turns, and the server accepts moves only from the player whose turn is active. A player selects a column with an available space, and their piece is placed in the lowest available position within that column.
- **Victory Condition:** A player wins the game by connecting four of their pieces consecutively in a horizontal, vertical, or diagonal line. After each move, the server checks the board to determine if a player has achieved four in a row.
- **Draw/Tie Condition:** The game ends in a draw if all positions on the 7x6 board are filled and neither player has connected four pieces. The server will notify both players that the game has resulted in a draw.

---

## 2. Application-Layer Messaging Protocol Blueprint (Sprint 1 Deliverable)

### 2.1 Message Transport & Serialization Format
- **Transport Protocol:** TCP
- **Serialization Format:** JSON
- **Framing Mechanism:** Newline-delimited JSON (`\n`). Each JSON message is encoded as UTF-8 and terminated with a newline character. The receiver uses a consistent stream buffer to handle TCP fragmentation and coalescing.

### 2.2 Message Schema Definitions

#### Message Types:
1. `CONNECT` (Client -> Server): Client requests to join the game with a player alias.
2. `LOBBY_WAIT` (Server -> Client): Server notifies Client 1 that it is waiting for Player 2 to connect.
3. `GAME_START` (Server -> Client): Server notifies both clients that the game has started and assigns each client a role, Player 1 or Player 2.
4. `MOVE` (Client -> Server): Active player submits a move to place a disc in a column.
5. `STATE_UPDATE` (Server -> Client): Server broadcasts the updated board state and active player's turn.
6. `ERROR` (Server -> Client): Server notifies the client of an out-of-turn move, invalid coordinates, or malformed message.
7. `DISCONNECT` (Client -> Server): Client notifies the server of an intentional departure or quit.
8. `GAME_OVER` (Server -> Client): Server broadcasts the final game outcome, including a win, draw, or forfeit.

#### Example JSON Protocol Schema:
```json
{
  "message": "MOVE",
  "player_id": "Hallie",
  "column": 4
}
```

---

### 2.3 Game State Machine (FSM) Design (Sprint 1 Deliverable)
- **State Transitions:** Detail state flow: `INIT` -> `WAITING_FOR_PLAYERS` -> `GAME_START` -> `PLAYER_TURN` -> `EVALUATE_MOVE` -> `CHECK_WIN_DRAW` -> `GAME_OVER` -> `CLEANUP` -> `INIT`.

---

## 3. Game Behavior & Server Concurrency Architecture (Sprint 2 Deliverable)

### 3.1 Server Concurrency Strategy
- **Architecture Choice:** [Multi-Threading (`threading.Thread`) OR Non-blocking I/O multiplexing (`select.select` / `selectors`)]
- **Synchronization Logic:** Explain how shared game state and client list are thread-safe (e.g. `threading.Lock`) to prevent race conditions during turn processing.

### 3.2 State & Score Synchronization Across Clients
- **Turn Enforcement:** Detail how the server validates active player ID before processing moves and broadcasts updated turn notifications to all clients.
- **Score & Board Synchronization:** Describe how state broadcasts keep client screens synchronized in real time.

---

## 4. Coding & AI Implementation Plan (Sprint 3)

- **Permitted AI Tools:** [e.g., GitHub Copilot, ChatGPT, Claude]
- **AI Prompting & Constraint Strategy:** Explain how you will constrain AI models to generate code (in Python or your chosen language) that adheres strictly to the protocol blueprint and FSM designed in Sprints 1 & 2.
- **Implementation Risk Management:** Detail your plan to leverage past programming experience and manage time to ensure code completion on schedule.

---

## 5. CML Multi-Subnet Topology & Wireshark Deployment Plan (Sprint 4 & 5 Deliverable)

> For now you can use the topology below. We may update this when we get to defining subnets.

### 5.1 Subnet & Router Design
- **Subnet A (Client 1):** `192.168.10.0/24` (Interface `Gi0/1` on Router R1)
- **Subnet B (Client 2):** `192.168.11.0/24` (Interface `Gi0/2` on Router R1)
- **Subnet C (Game Server):** `192.168.20.0/24` (Interface `Gi0/1` on Router R2)
- **Router Backbone:** `10.0.0.0/30` (Interface `Gi0/0` on R1 <-> `Gi0/0` on R2)

### 5.2 DHCP Pools & DNS Configuration Plan
- **Router R1 DHCP Pool 1 (`CLIENT1_POOL`):** Leases `192.168.10.10` - `192.168.10.50`, gateway `192.168.10.1`, DNS `10.0.0.2`.
- **Router R1 DHCP Pool 2 (`CLIENT2_POOL`):** Leases `192.168.11.10` - `192.168.11.50`, gateway `192.168.11.1`, DNS `10.0.0.2`.
- **Router R2 Authoritative DNS:** Configured with `ip dns server` and static host mapping `server.[yourlastname].edu` -> `192.168.20.100`.

### 5.3 Deployment Strategy & Wireshark Trace Capture
- **CML Deployment Strategy:** Deploy `server.py` onto Subnet C node (`192.168.20.100`) behind Router R2, and `client.py` onto Subnet A and Subnet B nodes behind Router R1.
- **Cisco Infrastructure Configuration:** Router R1 DHCP pools (`CLIENT1_POOL`, `CLIENT2_POOL`) and Router R2 authoritative DNS (`ip host server.[lastname].edu 192.168.20.100`).
- **Wireshark Trace Capture Plan:** Capture DHCP DORA exchange (`dhcp_negotiation.pcap`) and DNS query/response resolution (`dns_lookup.pcap`).
