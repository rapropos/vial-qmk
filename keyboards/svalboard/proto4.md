# Protocol v4 proposal

The most convenient upper bound for a structure is the hidraw limit of 
32 bytes, so these are designed to fit in one hidraw packet. All strings
are zero-terminated.

Command opcodes are assigned so that all but the least significant bit 
identify the category of data to be addressed, bit 0 clear indicates 
a read operation and bit 0 set is the corresponding write. IOW, reads are
even opcodes and adding 1 to the read opcode gives you the write one.

If there is no odd opcode, the data is read-only. For discussion
purposes, we start new opcodes at 0x20.

Unless otherwise noted, the response from the board to every push
request should be identical to what subsequent read requests for
the given entity would be until further pushes occur.

## Data type definitions

This document uses the nomenclature from keybard to describe structure
fields:

```
<, >: little and big endian
b, B: 1 byte
h, H: 2 bytes
i, I: 4 bytes
q, Q: 8 bytes
```

Lower-case indicates signed integers, upper-case are unsigned.
All fields must be naturally aligned, meaning that 16-bit 
quantities must be even-aligned, words must be word-aligned,
and dwords must be dword-aligned.

Since the protocol version was initially defined as little-endian,
all of the other >8 bit fields in this proposal are also little-endian.
This could be changed to use network order for everything but legacy
protocol version if desired.

## Ecosystem-wide information (opcodes 0x20-0x2F)

Communication that impacts the product as a whole, as opposed to
individual components like pointing devices.

### Protocol version, initial handshake (opcodes 0x20/0x21, legacy opcode 0x01)

caller sends:

```
type svalProtocolVersionReq {
    cmdfam:    B;   // 0xEE for Svalboard
    opcode:    B;   // 0x01 or 0x20
}
```

board returns:

```
type svalBannerRsp {
    svalMarker:         B[4];   // "sval"
    protocolVersion:    <I;     // if >= 4, following fields are present
    numPeripherals:     B;
    numLayers:          B;
    configFlags:        <H;
    reserved:           B[20];
}
```

configFlags can carry up to 16 configuration flags that apply to the board
as a whole, as opposed to an integrated peripheral. Obvious candidates 
would be things like achordion. If more than 16 flags are desired, numLayers
could be taken out, giving us another 8, or bytes 0x0a/0x0b could be used for
something else, moving settingFlags down to a dword starting at 0x0c.

opcode 0x20 is essentially a superset of opcode 0x01. Global setting flags
are set with:

pusher sends:

```
type svalGlobalConfigPush {
    cmdfam:         B;   // 0xEE for Svalboard
    opcode:         B;   // 0x21
    settingFlags:   <H;
}
```

gets back the same banner response as a future 0x20 would, incorporating
the new settings.

### Layout cosmetics (opcodes 0x22-0x25)

It can be useful for the UI to be able to tell the user which set of
global settings are currently active.

#### Layout names (opcodes 0x22/0x23)

seeker sends:

```
type svalLayoutNameReq {
    cmdfam:    B;       // 0xEE for Svalboard
    opcode:    B;       // 0x22
}
```

gets back:

```
type svalLayoutNameRsp {
    name:       B[28];
}
```

pusher sends:

```
type svalLayoutNamePush {
    cmdfam:     B;      // 0xEE for Svalboard
    opcode:     B;      // 0x23
    name:       B[28]; 
}
```

#### Layout revisions (opcodes 0x24/0x25)

seeker sends:

```
type svalLayoutRevisionReq {
    cmdfam:     B;       // 0xEE for Svalboard
    opcode:     B;       // 0x24
}
```

gets back:

```
type svalLayoutRevisionRsp {
    revision:   B[16];
    asof:       <Q;     // timestamp of some sort
}
```

Revisions are opaque, so that userspace can decide how to
store and present them. A version control commit slug would seem a
reasonable possibliity. ISO 8601 textual timestamps are potentially
so large that they would need a dedicated packet just to house one,
so some sort of monotonic count of seconds from some epoch would seem
a better fit here. If 8 bytes is enough for PostgreSQL, it should be
enough here.

pusher sends:

```
type svalLayoutRevisionPush {
    cmdfam:     B;      // 0xEE for Svalboard
    opcode:     B;      // 0x25
    revision:   B[16];
    asof;       <Q;
}
```

Opcodes 0x26-0x2F are reserved for future communication involving the
entire ecosystem.

## Peripherals (opcodes 0x30-0x3F)

Most standard Svalboards will have two peripherals - a left and right
pointing device. Assuming it would be theoretically possible to chain
further such peripherals, such as footpedals, joysticks, head-mounted
motion sensors, etc, it seems potentially useful to have these be a 
first-class citizen in the protocol. Peripheral slot numbers can either
be arbitrarily defined by assigning constants to right and left hand,
in which case there should probably be a flag indicating whether the
peripheral in question is on the driver side of the board or not, or
we could use slot 1 to always refer to the driver pointing device and
let everybody else come in afterwards.

### Peripheral model (opcode 0x30)

seeker sends:

```
type svalPeripheralModelReq {
    cmdfam:         B;      // 0xEE for Svalboard
    opcode:         B;      // 0x30
    slot:           B;      // <numPeripherals from board banner
}
```

gets back: 

```
type svalPeripheralModelRsp {
    driverName:     B[16];  // typically POINTING_DEVICE_DRIVER
}
```

Fields could be added here to identify capabilities of the peripheral;
for now this is assumed to be inferrable from the driver name. If that's
deemed sufficient, could extend driverName to permit up to 28 bytes.

As peripheral models are not modifiable from userspace, opcode 0x31 is
reserved and unused.

### Peripheral name (opcodes 0x32/0x33)

These can be assigned by the user and stored in .kbi files indexed by
slot. This would allow the UI to present them in a way so that the
user can easily distinguish things like left or right foot pedals.

seeker sends:

```
type svalPeripheralNameReq {
    cmdfam:         B;      // 0xEE for Svalboard
    opcode:         B;      // 0x32
    slot:           B;      // <numPeripherals from board banner
    reserved0:      B;
}
```

gets back:

```
type svalPeripheralNameRsp {
    slot:           B;
    name:           B[28];
}
```

pusher sends:

```
type svalPeripheralNamePush {
    cmdfam:         B;      // 0xEE for Svalboard
    opcode:         B;      // 0x33
    slot:           B;      // <numPeripherals from board banner
    reserved0:      B;
    name:           B[28];
}
```

Board responds to a push with the same NameRsp that would have come back for
a request for the name.

### Peripheral configuration (opcodes 0x34/0x35)

seeker sends:

```
struct svalPeripheralConfigReq {
    cmdfam:         B;      // 0xEE for Svalboard
    opcode:         B;      // 0x34
    slot:           B;      // <numPeripherals from board banner
}
```

gets back:

```
struct svalPeripheralConfigRsp {
    slot:           B;
    layer:          B;
    flags:          >H;
    cpiV:           >H;
    cpiH:           >H;
    decay:          >I;
}
```

layer is the dedicated layer that this peripheral may automatically activate.
The UI can use this information to warn or constrain the user about modifying
keys on layers that "belong" to integrated peripherals. If layer is 0xFF, the
peripheral is not capable of auto-activating layers. It probably makes sense
to leave layer on a valid value even if auto-activation is not currently
active, as that could be a transient runtime state, and the UI would still
want to know what layer the peripheral *might* grab.

flags gives us 16 bits for booleans such as "is enabled", "scroll locked",
"auto enable layer", &c.

cpiV and cpiH could be combined if desired, but it might make sense to permit
a device to see the world as a rectangle, not a square. The scale of these
fields is dependent on the peripheral type. For example, a PMW3389 sensor
could treat this as something to be multiplied by 50 to get actual CPI, as
this is the finest granularity supported by the hardware.

It could make sense to define decay as microseconds, with all bits set
as an indicator of infinity. This would be equivalent to the mh_timer 
settings in the firmware, and if 16 bits of precision are deemed sufficient,
this could be narrowed.

pusher sends:

```
struct svalPeripheralConfigReq {
    cmdfam:         B;      // 0xEE for Svalboard
    opcode:         B;      // 0x35
    slot:           B;      // <numPeripherals from board banner
    layer:          B;
    flags:          >H;
    cpiV:           >H;
    cpiH:           >H;
    decay:          >I;
}
```

If it's deemed impractical to allow runtime alteration of the layer
dedicated to a particular peripheral, layer could be removed and its
byte reserved as padding. Pushers are obligated to retain all bits
in flags that they are not actively altering. If this is deemed too
onerous of a requirement, an additional bitmask field could be added
advertise which bits in flags should actually be modified, but it seems
this would be easiest to just leave to userland.

The response to a push is identical to that which will be received
for subsequent read requests.

Opcodes 0x36 - 0x3F are reserved for additional integration with
integrated peripherals.

## Layer cosmetics (opcodes 0x40-0x4F)

Layers can have associated colors and user-assigned names that can be
edited at runtime. Colors are stored as byte-scaled HSV, with V channel
values of 0xFF meaning that the global backlight brightness should be
substituted. It would be possible to add a global config flag to direct
the firmware to attempt some sort of blended scaling of the explicitly
defined brightness of a layer color with the global brightness setting.

seeker sends:

```
struct svalLayerCosmeticReq {
    cmdfam:         B;      // 0xEE for Svalboard
    opcode:         B;      // 0x40
    layer:          B;      // <numLayers from board banner
}
```

gets back:

```
struct svalLayerCosmeticRsp {
    layer:          B;
    hue:            B;
    sat:            B;
    val:            B;      // color brightness
    name:           B[26];
}
```

pusher sends:

```
struct svalLayerCosmeticPush {
    cmdfam:         B;      // 0xEE for Svalboard
    opcode:         B;      // 0x41
    layer:          B;      // <numLayers from board banner
    hue:            B;
    sat:            B;
    val:            B;      // color brightness
    name:           B[26];
}
```

Opcodes 0x42-0x4F are reserved for future use involving layer-specific communication.
All opcodes above 0x4F are reserved for future use.