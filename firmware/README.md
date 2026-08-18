# LockStep firmware

Two firmware images per node — see `docs/DESIGN.md` §2 for the split-brain rationale and
`docs/pin-map.html` for every pin (verified against the ST pin database + AN2606 Rev 61).

## `lockstep-node/` — STM32G491 motion firmware (STM32CubeIDE)

`lockstep-node.ioc` carries the full verified pin map with net labels (open it in
CubeIDE ≥ 2.2 and generate). Clock tree is pre-set: 16 MHz HSE → PLL → 170 MHz core,
HSI48 + CRS → USB. A few one-click settings are left for CubeMX (they have no stable
.ioc keys worth hand-authoring): USART2 hardware flow control (RTS/CTS), SPI2 →
*Receive Only Master* (MT6701 SSI), USB → *Device (CDC)*, and EXTI enables on
DIAG0/DIAG1/ENC_Z.

Planned module layout (all sit on HAL/LL, no RTOS decision yet — bring-up starts
bare-metal superloop + SysTick):

| Module | Job |
|---|---|
| `motion/` | TMC5130 driver (SPI), ramp-target control, closed-loop correction from MT6701, IRUN derating from the PD budget |
| `encoder/` | MT6701 SSI reads, zero-offset, multi-turn tracking |
| `bus/` | FDCAN2 setup (1 M arb / 2–5 M data), node addressing (NODE_ID straps), setpoint + sync consumption, coordinator election (config-ID first) |
| `coord/` | coordinator role: program storage (A/B-aware flash layout), IK/blending/S-curve interpolation, setpoint streaming |
| `link/` | framed UART protocol to the C6 (3 Mbaud, CTS/RTS, CRC), C6 boot/reset strap control, esptool passthrough mode |
| `pd/` | AP33772S I2C: read granted PDO, publish the power budget, PD_INT handling |
| `dfu/` | A/B image slots + CAN-FD DFU receiver; C6→STM32 recovery path relies on ROM (USART2, 8-E-1) — no code needed here |
| `cmd/` | one command set, three transports: USB CDC, CAN-FD, link — same parser |

## `c6-radio/` — ESP32-C6 wireless firmware (ESP-IDF, not started)

Pure peripheral: WiFi 6 / BLE provisioning, wireless transport bridging to the link
UART, self-OTA, and pushing STM32 images via the ROM bootloader (drive `GPIO2` →
BOOT0, `GPIO3` → NRST, then STM32 ROM protocol on UART0 at 8-E-1).

## Flashing / recovery matrix

| Target | Bench | Field | Last resort |
|---|---|---|---|
| STM32 | SWD (Tag-Connect) | USB DFU · CAN-FD DFU (A/B) | C6 drives BOOT0+NRST → ROM UART |
| C6 | esptool via STM32 passthrough | WiFi self-OTA | STM32 drives EN+BOOT → ROM UART |
