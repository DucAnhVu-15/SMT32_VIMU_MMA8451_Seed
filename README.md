# VIMU UART Message Protocol

## 1. General frame

|B01 |B02 |B03 |B04 |B05  |B06  |B07..<B(n-2)>|B(n-1)|Bn   |
|:--:|:--:|:--:|:--:|:---:|:---:|:-----------:|:----:|:---:|
|0xAA|0x55|Type|Cmd |Len_L|Len_H|Payload      |CRC_L |CRC_H|

> B01 + B02: Start of frame (fixed 0xAA 0x55)

> Type: Packet type

> Cmd: Command code

> Len_L + Len_H: Payload length in bytes (little-endian)

> Payload: Data field, variable length

> CRC_L + CRC_H: 16-bit CRC, transmit low byte first

### Length rule

- `payload_length = Len_L + (Len_H << 8)`
- `frame_length = 8 + payload_length`

---

## 2. Packet type

|Type|Name|Direction|Description|
|:--:|:---|:--------|:----------|
|0x01|CMD |PC -> VIMU|Command message|
|0x02|RSP |VIMU -> PC|Response message|
|0x03|EVT |VIMU -> PC|Event push message|
|0x04|ACK |VIMU -> PC|Short acknowledge|
|0x05|NACK|VIMU -> PC|Reject + error code|

---

## 3. Command code

|Cmd|Name|Role|
|:--:|:---|:---|
|0x10|HANDSHAKE|Link check / sync session|
|0x11|NEW_DATA|Send new payload block|
|0x12|CLEAR|Clear buffer / data area|
|0x13|STOP|Stop processing|
|0x14|START|Start processing|
|0x15|UPDATE|Apply update request|
|0x20|REQ_MORE_DATA|Request additional data block|
|0x21|STATUS_PUSH|Push status from device|

---

## 4. Full message list (Send/Response)

### 4.1 Messages sent from PC to VIMU

|Direction|Type|Cmd|Name|Description|
|:--------|:---:|:--:|:---|:----------|
|PC -> VIMU|0x01|0x10|CMD/HANDSHAKE|Link check / sync session|
|PC -> VIMU|0x01|0x11|CMD/NEW_DATA|Send payload block|
|PC -> VIMU|0x01|0x12|CMD/CLEAR|Clear buffer or data area|
|PC -> VIMU|0x01|0x13|CMD/STOP|Stop processing|
|PC -> VIMU|0x01|0x14|CMD/START|Start processing|
|PC -> VIMU|0x01|0x15|CMD/UPDATE|Apply update request|

### 4.2 Response messages from VIMU to PC

|Direction|Type|Cmd|Name|Description|
|:--------|:---:|:--:|:---|:----------|
|VIMU -> PC|0x02|0x10..0x15|RSP/*|Response for command packets|
|VIMU -> PC|0x04|Same as request|ACK|Short acknowledge|
|VIMU -> PC|0x05|Same as request|NACK|Reject + error code in payload|

### 4.3 Event push messages from VIMU to PC

|Direction|Type|Cmd|Name|Description|
|:--------|:---:|:--:|:---|:----------|
|VIMU -> PC|0x03|0x20|EVT/REQ_MORE_DATA|Request additional data block|
|VIMU -> PC|0x03|0x21|EVT/STATUS_PUSH|Push status to host|

---

## 5. Error code (for NACK payload)

|Code|Name|Meaning|
|:--:|:---|:------|
|0x00|VIMU_UART_ERR_NONE|No error|
|0x01|VIMU_UART_ERR_BAD_CRC|CRC invalid|
|0x02|VIMU_UART_ERR_BAD_LENGTH|Length invalid|
|0x03|VIMU_UART_ERR_UNKNOWN_CMD|Unknown command|
|0x04|VIMU_UART_ERR_INVALID_STATE|Invalid state|
|0x05|VIMU_UART_ERR_BUFFER_FULL|Buffer full|
|0x06|VIMU_UART_ERR_BUFFER_EMPTY|Buffer empty|
|0x07|VIMU_UART_ERR_BAD_FORMAT|Frame format invalid|
|0x08|VIMU_UART_ERR_UNSUPPORTED|Unsupported feature|

---

## 6. Status values

### System state

|Value|Name|
|:---:|:---|
|0x00|VIMU_STATE_STOPPED|
|0x01|VIMU_STATE_RUNNING|

### Buffer level

|Value|Name|
|:---:|:---|
|0x00|VIMU_BUF_EMPTY|
|0x01|VIMU_BUF_1S|
|0x02|VIMU_BUF_2S|
|0x03|VIMU_BUF_3S|
|0x04|VIMU_BUF_4S|
|0x05|VIMU_BUF_FULL|

At 50 Hz, `1S` is 50 samples. The STM32 ring buffer stores 500 samples
(10 seconds), and `VIMU_FLAG_BUFFER_LOW` is asserted when the ring has fewer
than 150 samples (below 3 seconds).

### Status flags (bit mask)

|Bit|Mask|Name|
|:--:|:--:|:---|
|0|0x01|VIMU_FLAG_BUFFER_LOW|
|1|0x02|VIMU_FLAG_FIFO_FULL|
|2|0x04|VIMU_FLAG_IRQ_ACTIVE|
|3|0x08|VIMU_FLAG_I2C_ENABLE|
|4|0x10|VIMU_FLAG_RING_EMPTY|

---

## 7. Full frame templates

### 7.1 CMD frames (PC -> VIMU)

#### Commands without payload

|Name|B01|B02|B03|B04|B05|B06|B07|B08|
|:---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|CMD/HANDSHAKE|0xAA|0x55|0x01|0x10|0x00|0x00|CRC_L|CRC_H|
|CMD/CLEAR|0xAA|0x55|0x01|0x12|0x00|0x00|CRC_L|CRC_H|
|CMD/STOP|0xAA|0x55|0x01|0x13|0x00|0x00|CRC_L|CRC_H|
|CMD/START|0xAA|0x55|0x01|0x14|0x00|0x00|CRC_L|CRC_H|
|CMD/UPDATE|0xAA|0x55|0x01|0x15|0x00|0x00|CRC_L|CRC_H|

#### CMD/NEW_DATA (payload variable)

|B01|B02|B03|B04|B05|B06|B07..B(6+N-1)|B(7+N)|B(8+N)|
|:--:|:--:|:--:|:--:|:--:|:--:|:-----------:|:-----:|:-----:|
|0xAA|0x55|0x01|0x11|Len_L|Len_H|Payload (N bytes)|CRC_L|CRC_H|

Example with N = 4:

|B01|B02|B03|B04|B05|B06|B07|B08|B09|B10|B11|B12|
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|0xAA|0x55|0x01|0x11|0x04|0x00|D0|D1|D2|D3|CRC_L|CRC_H|

### 7.2 Response frames (VIMU -> PC)

#### RSP (generic response packet)

|B01|B02|B03|B04|B05|B06|B07..B(6+M-1)|B(7+M)|B(8+M)|
|:--:|:--:|:--:|:--:|:--:|:--:|:-----------:|:-----:|:-----:|
|0xAA|0x55|0x02|Cmd|Len_L|Len_H|Payload (M bytes)|CRC_L|CRC_H|

#### ACK (short acknowledge)

|B01|B02|B03|B04|B05|B06|B07|B08|
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|0xAA|0x55|0x04|Cmd|0x00|0x00|CRC_L|CRC_H|

#### NACK (reject + error code)

|B01|B02|B03|B04|B05|B06|B07|B08|B09|
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|0xAA|0x55|0x05|Cmd|0x01|0x00|ErrorCode|CRC_L|CRC_H|

Example NACK for BAD_CRC (0x01):

|B01|B02|B03|B04|B05|B06|B07|B08|B09|
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|0xAA|0x55|0x05|0x11|0x01|0x00|0x01|CRC_L|CRC_H|

### 7.3 EVT frames (VIMU -> PC, push)

#### EVT/REQ_MORE_DATA (0x20)

|B01|B02|B03|B04|B05|B06|B07|B08|
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|0xAA|0x55|0x03|0x20|0x00|0x00|CRC_L|CRC_H|

#### EVT/STATUS_PUSH (0x21)

|B01|B02|B03|B04|B05|B06|B07|B08|B09|B10|B11|B12|B13|B14|B15|
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|0xAA|0x55|0x03|0x21|0x07|0x00|State|BufLevel|Flags|Ring_L|Ring_H|Free_L|Free_H|CRC_L|CRC_H|

Status payload fields:

|Offset|Name|Size|Description|
|:--:|:---|:--:|:---|
|0|State|1|System state|
|1|BufLevel|1|Buffer level enum|
|2|Flags|1|Status flags|
|3|RingSamples|2|Queued samples in ring buffer, little-endian|
|5|FreeSamples|2|Samples needed to fill ring to 500, little-endian|

Note:
- Cmd in ACK/NACK/RSP should echo the related request command.
- CRC depends on the full bytes from B01 to the last payload byte.

---

## 8. Typical communication flow

1. PC send `CMD/HANDSHAKE` to establish session.
2. PC can send `CMD/CLEAR` to reset previous data.
3. PC send `CMD/START`.
4. PC sends one or many `CMD/NEW_DATA` packets. For reliable streaming the
   payload is `sequence_le16 + N * 6-byte samples`; retrying the same sequence
   is idempotent. The legacy `N * 6-byte samples` payload remains accepted.
5. Device may send `EVT/REQ_MORE_DATA` or `EVT/STATUS_PUSH` while running.
6. PC send `CMD/STOP` when done.
7. PC send `CMD/UPDATE` if update/apply action is required.
8. Device returns `ACK` on success or `NACK + error_code` on failure.

The STM32 independent watchdog is enabled with an approximately 4-second
timeout. A peripheral interrupt storm or other unexpected main-loop stall
therefore causes an automatic reboot; the PC should re-establish HANDSHAKE.

Protocol-v3 status payloads append byte 7 (`fw_proto=3`). The ESP32 ULP checks
the emulated `CTRL_REG1.ACTIVE` bit once per second; if the STM32 watchdog has
reset, it wakes the ESP32 main CPU so the sensor registers are configured again.

---

## 9. Notes

- CRC algorithm must be consistent on both PC and VIMU sides.
- Multi-byte numeric fields use little-endian order.
- Maximum payload is 1024 bytes.
- Recommended host refill policy: when `VIMU_FLAG_BUFFER_LOW` is set or
  `EVT/REQ_MORE_DATA` is received, send `FreeSamples` from the latest status
  payload so the ring returns to 500 samples.

---

## 10. I2C slave map for ESP32

- STM32 acts as `I2C slave` on `I2C1`
- Default 7-bit address: `0x42`
- Pins: `PB6 = SCL`, `PB7 = SDA`
- IRQ output to MCU: `PA8` active-high pulse when a FIFO block becomes ready
- FIFO is filled at `50 Hz`, one sample per tick, until it reaches `32 samples`
- When FIFO is full, STM32 raises an `IRQ` pulse, freezes the block, and lets ESP32 read all `192 bytes`

### 10.1 Read registers

|Reg|Name|Size|Description|
|:--:|:---|:--:|:----------|
|0x00|STATE|1|`0x00 = STOPPED`, `0x01 = RUNNING`|
|0x01|BUFFER_LEVEL|1|Same enum as UART status payload|
|0x02|FLAGS|1|Same bitmask as UART status payload|
|0x03|FIFO_SAMPLE_COUNT|1|Current queued sample count in FIFO|
|0x04|RING_SAMPLE_COUNT|1|Queued sample count remaining in ring buffer|
|0x05|TOTAL_SAMPLE_COUNT|1|`ring + fifo` sample count|
|0x06|FIFO_BLOCK_READY|1|`1 = 32-sample block ready for MCU read`|
|0x07|FIFO_BLOCK_BYTES|1|Always `192` bytes|
|0x08|PROTOCOL_VERSION|1|Current I2C map version = `0x02`|
|0x40..0xFF|FIFO_BLOCK_DATA|192|Full FIFO block `32 samples x 6 bytes`|

### 10.2 Write register

|Reg|Value|Meaning|
|:--:|:---:|:------|
|0x20|0x01|START filling FIFO at 50 Hz|
|0x20|0x02|STOP filling FIFO|
|0x20|0x03|CLEAR ring + FIFO + IRQ|
|0x20|0x04|Release current 32-sample block after MCU is done reading|

### 10.3 Suggested ESP32 read flow

1. Wait for external `IRQ` pulse from STM32 on `PA8`.
2. Read register `0x06`; if value is `1`, a full 32-sample block is ready. This register stays `1` after the IRQ pulse until the block is released.
3. Write register `0x40`, then read `192` bytes in one burst.
4. After the read completes, STM32 clears that FIFO block automatically.
