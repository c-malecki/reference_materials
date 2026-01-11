```c
static inline esp_err_t dht_fetch_data(dht_sensor_type_t sensor_type, gpio_num_t pin, uint8_t data[DHT_DATA_BYTES])
{
    uint32_t low_duration;
    uint32_t high_duration;

    // Phase 'A' pulling signal low to initiate read sequence
    gpio_set_direction(pin, GPIO_MODE_OUTPUT_OD);
    gpio_set_level(pin, 0);
    ets_delay_us(sensor_type == DHT_TYPE_SI7021 ? 500 : 20000);
    gpio_set_level(pin, 1);

    // Step through Phase 'B', 40us
    CHECK_LOGE(dht_await_pin_state(pin, 40, 0, NULL),
               "Initialization error, problem in phase 'B'");
    // Step through Phase 'C', 88us
    CHECK_LOGE(dht_await_pin_state(pin, 88, 1, NULL),
               "Initialization error, problem in phase 'C'");
    // Step through Phase 'D', 88us
    CHECK_LOGE(dht_await_pin_state(pin, 88, 0, NULL),
               "Initialization error, problem in phase 'D'");

    // Read in each of the 40 bits of data...
    for (int i = 0; i < DHT_DATA_BITS; i++)
    {
        CHECK_LOGE(dht_await_pin_state(pin, 65, 1, &low_duration),
                   "LOW bit timeout");
        CHECK_LOGE(dht_await_pin_state(pin, 75, 0, &high_duration),
                   "HIGH bit timeout");

        uint8_t b = i / 8;
        uint8_t m = i % 8;
        if (!m)
            data[b] = 0;

        data[b] |= (high_duration > low_duration) << (7 - m);
    }

    return ESP_OK;
}
```

|= operator:
Bitwise OR assignment. data[b] |= value means data[b] = data[b] | value
The bit shift expression:
cdata[b] |= (high_duration > low_duration) << (7 - m);
```

Breaking it down:

1. `(high_duration > low_duration)` evaluates to `1` (true/bit set) or `0` (false/bit clear)
2. `<< (7 - m)` shifts that bit left by `(7 - m)` positions
3. `|=` ORs it into `data[b]`, setting that specific bit

**Example for byte 0:**
```
i=0: m=0, shift left 7 → sets bit 7 (MSB): 10000000
i=1: m=1, shift left 6 → sets bit 6:      01000000
i=2: m=2, shift left 5 → sets bit 5:      00100000
...
i=7: m=7, shift left 0 → sets bit 0 (LSB): 00000001
Why this works:
DHT sends 40 bits (5 bytes). Each bit is encoded as pulse duration - longer pulse = 1, shorter = 0. The code:

Reads bits MSB-first (bit 7 down to bit 0 for each byte)
Compares pulse durations to determine if bit is 1 or 0
Shifts the bit to correct position
ORs it into the byte

So after 8 iterations, data[0] contains the first byte with all 8 bits set correctly.

```c
/**
 * Pack two data bytes into single value and take into account sign bit.
 */
static inline int16_t dht_convert_data(dht_sensor_type_t sensor_type, uint8_t msb, uint8_t lsb)
{
    int16_t data;

    if (sensor_type == DHT_TYPE_DHT11)
    {
        data = msb * 10;
    }
    else
    {
        data = msb & 0x7F;
        data <<= 8;
        data |= lsb;
        if (msb & BIT(7))
            data = -data;       // convert it to negative
    }

    return data;
}
```

Temperature: -12.3°C → raw value 123

msb = 10000111 (bit 7 set = negative, bits 0-6 = 0x07)
lsb = 10111011 (0xBB = 187 decimal)

data = 0x07 << 8 | 0xBB = 0x07BB = 1979
Check bit 7 of original msb → set, so negate: -1979
But wait, let me recalculate...

Actually: 12.3°C = 123 tenths
msb = 0x00, lsb = 0x7B → 0x007B = 123 ✓
For -12.3°C: msb = 0x80 | 0x00 = 0x80, lsb = 0x7B
