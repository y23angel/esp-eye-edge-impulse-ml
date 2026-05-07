# 02. UART & AT Commands

**Files**: `main/main.cpp`, `firmware-sdk/at-server/ei_at_server.cpp`, `edge-impulse/.../ei_at_handlers.cpp`

## Full Flow

```
User types: "AT+RUNIMPULSE\r"
                │
                ▼
ei_get_serial_byte()           // wraps getchar()
                │              // \n → \r conversion
                │              // returns 0xFF when no data (EOF = -1 as char)
                ▼
ATServer::handle(char c)
  ├── normal chars  → buffer.add(c)        // accumulate command string
  ├── 0x1b (ESC)   → control_sequence[]   // arrow keys, HOME, END
  └── '\r'          → execute("AT+RUNIMPULSE")
                │
                ▼
ATServer::execute(input)
  ├── parser.parse()            // extract command name
  └── registered_commands[]    // find & call matching handler
```

## buffer vs control_sequence

| Storage | Role |
|---------|------|
| `buffer` | Accumulates the typed command string until `\r` |
| `control_sequence` | Temporary storage for escape sequences (arrow keys etc.), cleared after use |

## AT Commands for Inference

| Command | continuous | debug | Behavior |
|---------|-----------|-------|----------|
| `AT+RUNIMPULSE` | false | false | Single run, 2 s delay between frames |
| `AT+RUNIMPULSECONT` | true | false | Continuous, no delay |
| `AT+RUNIMPULSEDEBUG=n` | false | true | Single run + base64 JPEG over UART |
| `AT+RUNIMPULSEDEBUG=y` | false | true | Same + UART bumped to 1 Mbps |

All three call `ei_start_impulse(continuous, debug, use_max_uart_speed)`.
