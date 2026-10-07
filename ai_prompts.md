# AI Prompts & Constraint Strategy

## Protocol Implementation Prompt

Implement the client/server message parser and serializer according to the specifications outlined in `protocol_blueprint.md`.

Adhere strictly to the blueprint:
- Use the specified message types, field names, and data types exactly as defined.
- Do not add, remove, or rename any fields.
- Utilize JSON over TCP with newline-delimited (`\n`) framing.
- Handle TCP fragmentation and coalescing using a persistent receive buffer.
- Validate messages against the schema provided in the blueprint.
- Handle malformed messages and TCP disconnects without causing the server to crash.

Do not make assumptions or introducing any protocol features that are not defined in `protocol_blueprint.md`.
