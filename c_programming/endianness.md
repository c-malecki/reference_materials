## Endianness

Endianness is the byte order in memory

Big-endian has the most significant byte first
Memory: [0x12][0x34][0x56][0x78]
         MSB              LSB

Little-endian has the least significant byte first
Memory: [0x78][0x56][0x34][0x12]
         LSB              MSB

Big-endian = reading left-to-right (English)
Little-endian = reading right-to-left (like writing the number backwards in memory)

Byte significance is which byte has the most impact on the value
e.g.
regular integer: 1234

Changing 1 to 9 is a big change 9234
Changing 4 to 9 is a small change 1239

ARM architecture is little-endian
Network protocols are usually big-endian
BLE addresses are stored little-endian but displayed big-endian so they often have to be reversed for display