# Changelog

## 4.5.1

Fix MQTT availability topic mismatch causing permanent entity unavailability.

### Bug fix
- Fixed `availability.topic` in `_send_discovery` and `_send_discovery_custom` — was using `{base_topic}/status` (resolves to `Cosy HP/status`) instead of the correct `qsh_modbus/status` where LWT and gateway status are actually published
- All sensor entities now correctly track the gateway online/offline status

### Housekeeping
- `sw_version` in discovery payloads updated from 4.5.0 to 4.5.1

## 4.5.0

Stale data protection and gateway connectivity status.

### MQTT discovery: expire_after
- All sensor discovery payloads now include `expire_after: 60` — Home Assistant marks entities unavailable after 60s of no updates, preventing stale data from appearing live during gateway outages

### Gateway connectivity status topic
- New `qsh_modbus/status` topic publishes `online`/`offline` (retained) reflecting Waveshare TCP socket state
- MQTT Last Will and Testament (LWT) ensures broker publishes `offline` if the sniffer process dies unexpectedly
- Explicit `offline` published when entering TCP reconnect loop; `online` restored on successful reconnect
- New `binary_sensor.modbus_gateway_status` (device_class: connectivity, entity_category: diagnostic) enables HA automations to detect prolonged outages and trigger remedial action (e.g. smart plug power-cycle)

### Housekeeping
- `sw_version` in discovery payloads updated from 4.3.0 to 4.5.0

## 4.4.0

Major register map update validated by two confirmed defrost events (2026-03-26 04:22 and 07:16 UTC) plus multiple reversing-valve actuations.

### Register renames (defrost validated)
- **Reg 29**: "HP LWT" → "Flow Temp"
- **Reg 30**: "Condenser Temp" → "Return Temp" (correlates with return pipe sensor)
- **Reg 38**: "Internal Unit Temp" → "Suction Line Temp" (wide range -13 to +27°C)
- **Reg 39**: "Outdoor Ambient Temp" → "Outdoor Coil Air Temp" (tracks below outdoor, swings during defrost)
- **Reg 40**: "System Return Temp" → "Condenser Outlet Temp" (water return path, always below flow)
- **Reg 41**: "T6 Sump" → "Evaporator Inlet Temp" (goes to -10°C during defrost, R290 cross-check)
- **Reg 43**: "T8 Liquid" → "Liquid Line Temp"
- **Reg 44**: "T9 Flow Temp" → "Condenser Mid Temp"
- **Reg 45**: "DHW Cylinder Temp" → "Compressor Shell Temp" (52-54°C steady, drops 18°C during defrost)
- **Reg 32**: "V1 Heating" → "V1 Heating Valve"
- **Reg 34**: "V3 Defrost" → "Reversing Valve" (100% heating, drops to 0% during defrost)
- **Reg 36**: "T1 External Temp" → "OAT External"
- **Reg 37**: "T2 Intermediate" → "Indoor Ambient"
- **Reg 27**: "Electrical Power In" → "Compressor Power"

### Critical correction: R64
- **Reg 64**: "Heat Output" → "State Accumulator" — wild uint16 swings during defrost prove this is NOT thermal output. R25 is the actual heat output register.

### New registers (previously published as "Modbus Reg XX")
- **Reg 19**: Compressor Frequency (scale ×0.1 Hz, confirmed by physics)
- **Reg 24**: Suction Pressure (scale ×0.1 kPa, R290 saturation cross-check)
- **Reg 25**: Heat Output (primary thermal measurement)
- **Reg 26**: Electrical Power Total (includes fans/pumps, ~150W above R27)
- **Reg 28**: Heat Output 2 (secondary heat metric)
- **Reg 55**: Evaporator Temp (primary frost signal, defrost validated)
- **Reg 56**: Discharge Temp (defrost validated)
- **Reg 57**: Defrost Accumulator (signed integrator, not countdown)
- **Reg 62**: EEV Opening (54-75% heating, 0% during defrost)
- **Reg 65**: Operating Mode (state machine: 2=heating, 6=pre-defrost, 7=defrost, 8=recovery)
- **Reg 20**: Runtime Counter
- **Reg 63**: Energy Counter (was "Unknown 63")
- **Reg 53**: DHW Tank Temp (constant 60.0°C, was misidentified as Compressor Frequency)
- **Reg 54**: Outdoor Ambient Raw (unsigned, no device_class — suspect packed register)
- **Reg 67, 75, 77**: Tracked for future analysis

### Removed registers
- **Reg 51**: Compressor Speed % — removed (R19 is the confirmed compressor register)
- **Reg 66**: Operating Mode — removed (R65 is the confirmed state machine)

### Demoted to Unknown
- **Reg 48**: "Discharge Pressure" → "Unknown 48" (plausible but not closed)
- **Reg 50**: "Reported COP" → "Unknown 50" (may not be a pressure)

### SIGNED_REGISTERS changes
- Added R19 (compressor Hz), R24 (suction pressure)
- Removed R54 (scaling suspect during defrost, published raw unsigned)

### HA entity impact
- Existing entities retain history (unique_id unchanged)
- Scale changes cause expected discontinuities for R19, R24, R55, R54
- R64 name change from "Heat Output" → "State Accumulator" will affect dashboards using friendly name

## 4.3.0

- **Reg 29**: Renamed "Flow Temp" → "HP LWT" — condenser outlet leaving water temp; reads 2–4°C above system flow before volumiser mixing
- **Reg 30**: Renamed "Return Temp" → "Condenser Temp" — condenser plate / refrigerant-side temp, NOT a water circuit measurement. Reads 40°C when off (retained heat), drops during operation, inverts vs reg 40
- **Reg 40**: Renamed "Return Water Temp" → "System Return Temp" — actual system EWT; tracks Shelly return within −0.4°C. Upgraded from STATISTICAL to CONFIRMED

## 4.2.0

- **Reg 45**: Remapped from "Discharge Gas Temp" to "DHW Cylinder Temp" (CONFIRMED from 2026-02-18 DHW heating cycle — 52–55°C range consistent with cylinder storage)
- **Regs 0/1**: Annotated as unresolved — always {0,0} during both space heat and DHW; may be write-once demand latch whose transition was not captured
- **Reg 92**: Added annotation — always 4 across both modes; likely operating mode flag, pending transition capture
- Wrapped TCP keepalive tuning (TCP_KEEPIDLE/INTVL/CNT) in try/except AttributeError for platform safety
- Changed watchdog log interval from 60s to 5 minutes with frame count reporting
- Updated socket timeout log message for consistency

## 4.1.1

- Fixed half-open TCP socket hang: recv() would block indefinitely when the Waveshare gateway connection dropped silently, causing hours of undetected data loss
- Added TCP keepalive (KEEPIDLE=30s, KEEPINTVL=10s, KEEPCNT=3) to detect dead connections at the OS level within ~60s
- Increased recv timeout to 30s and wired it into reconnection logic (hub sends every ~2.5s, so 30s silence = dead connection)
- Added periodic watchdog log line ("Recv loop alive") every 60s to make future hangs immediately visible in logs

## 4.1.0

- Added register identifications from statistical analysis of HA history data (24h, ~3s interval)
- **High confidence**: reg_38 Internal Unit Temp, reg_39 Outdoor Ambient Temp, reg_40 Return Water Temp, reg_45 Discharge Gas Temp, reg_50 Reported COP, reg_51 Compressor Speed %, reg_53 Compressor Frequency Hz, reg_66 Operating Mode
- **Medium confidence** (documented in unknown.md): reg_26 alt power, reg_56 alt heat output, reg_60 condensing pressure, reg_65 month
- Corrected reg_50 from "Suction Pressure" to "Reported COP" (mean 4.23, cross-validated within 7% of implied COP)
- Corrected reg_38 from "T3 Suction" to "Internal Unit Temp" (tracks outdoor +5°C, r=0.953)
- Corrected reg_39 from "Evaporator Temp" to "Outdoor Ambient Temp" (range matches UK ambient)
- Added identified.md and unknown.md register documentation

## 4.0.0

- Fixed Modbus framing: scanning parser extracts multiple frames from concatenated TCP recv() chunks via CRC probing (fixes 0% CRC pass rate)
- Added pair_response() on ModbusFrame for clean request/response register mapping
- Added FC 0x05 (Write Single Coil) and FC 0x0F (Write Multiple Coils) parsing
- Upgraded OperatingStateDetector from timing-based (ACTIVE/IDLE/HEARTBEAT) to register-based (OFF/DEFROST/DHW/HEATING/HEATING_IDLE/OIL_RECOVERY) with state history and trigger register logging
- Upgraded RegisterTracker with min/max values, sample counts, function code tracking, and write register tracking
- Added MQTT state transition publishing and coil publishing
- Added MQTT discovery for unknown registers (as "Modbus Reg XX")
- Added debug logging of raw recv() hex data (enable with --debug or DEBUG=true)

## 3.0.0

- Migrated from standalone script to Home Assistant add-on
- Added S6 process supervision with automatic restart on crash
- Added exponential backoff reconnection to Waveshare gateway
- Added RotatingFileHandler (10MB × 5 backups) to prevent log disk fill
- Added MQTT auto-discovery from HA Supervisor API (no manual credentials needed)
- Added reconnect counter to stats logging
- Handles ConnectionResetError, BrokenPipeError, OSError gracefully
- Daily CSV log rotation

## 2.1.0

- Added reg_27 as "Electrical Power In" (CONFIRMED r=0.999 vs Shelly EM)
- Fixed reg_64 from "Energy Elec Consumed" to "Heat Output" (CONFIRMED r=0.999 vs flow×ΔT)
- Parked reg_63 as "Unknown 63" pending verification

## 2.0.0

- Complete rewrite with raw register naming (reg_XX)
- Added confidence annotations (CONFIRMED / NAMED / UNCONFIRMED)
- Added signed int16 handling for temperature registers
- Added operating state detection (ACTIVE / HEARTBEAT / IDLE)
- Added CSV frame logging

## 1.0.0

- Initial passive sniffer with hardcoded register names
- MQTT publishing with HA auto-discovery
- Waveshare RS485-to-WiFi gateway support
