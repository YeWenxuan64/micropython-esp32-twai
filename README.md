# micropython-esp32-twai

This repository adds TWAI/CAN(USER_C_MODULES) support to MicroPython for the ESP32 family.
Use user_module, that draft probe. But works.
From:
https://github.com/micropython/micropython/pull/12331
https://github.com/straga/micropython-esp32-twai

YeWenxuan: I improved this repository to support MicroPython v1.27+

## Prepare to build

Read this section if you want to include the ESP32 TWAI/CAN support to MicroPython from scratch. To do that follow these steps:

0. Make a work directory then enter it:
    ```bash
    mkdir -p ~/esp
    cd ~/esp
    ```

1. Clone the esp-idf repository:
    ```bash
    git clone -b v5.5.1 https://github.com/espressif/esp-idf.git
    cd esp-idf
    git submodule update --init --recursive --force
    cd ..
    ```

2. Clone the MicroPython repository:
    ```bash
    git clone --recursive https://github.com/micropython/micropython.git
    cd micropython
    git submodule update --init --recursive --force
    cd ..
    ```
  
3. Clone this repository:
    ```bash
    git clone https://github.com/YeWenxuan64/micropython-esp32-twai.git
    ```

4. Check the directory structure which should look like this:
    ```
    /                      # now we are here
    ├──esp-idf/            # esp-idf repository
    │
    ├── micropython/       # MicroPython repository
    │   └── port/
    │       └── esp32/     # build point
    │
    └── micropython-esp32-twai/
        └── src_can_v2/
            └── micropython.cmake
    ```

## Build

### SETUP ESP-IDF

```bash
# Go to esp-idf repository
cd esp-idf

# Install the esp-idf environment and export
./install.sh
. ./export.sh
cd ..
```

### Build MicroPython

```bash
# Go to micropython repository
cd micropython

# Build mpy-cross
make -C mpy-cross

# Go to ESP32 port
cd ports/esp32

# Update submodules
make submodules

# e.g. if you are using esp32-s3
# make BOARD=ESP32_GENERIC_S3 submodules

# e.g. if you are using esp32-s3 with octal PSRAM
# make BOARD=ESP32_GENERIC_S3 BOARD_VARIANT=SPIRAM_OCT submodules
```

### Build with TWAI/CAN Support
***note* some arguments need to be filled in truthfully, like the size of flash**

```bash
# -D MICROPY_BOARD= `your esp board`
# -D MICROPY_BOARD_VARIANT=SPIRAM_OCT `if your esp32-s3 has octal PSRAM`
# -D CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y `if your esp32-s3 has 16 MB flash, like N16R8`

idf.py -D MICROPY_BOARD=ESP32_GENERIC_S3 -D MICROPY_BOARD_VARIANT=SPIRAM_OCT -D CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y -D USER_C_MODULES="../../../../micropython-esp32-twai/src_can_v2/micropython.cmake" build
```


## Usage Example

For testing, connect pin 4 and pin 5 together for loopback mode.

```python
import asyncio
import CAN

dev = CAN(0, extframe=False, tx=5, rx=4, mode=CAN.LOOPBACK, bitrate=50000, auto_restart=False)


# - identifier of can packet (int)
# - extended packet (bool)
# - rtr packet (bool)
# - data frame (0..8 bytes)

async def reader():
    while True:
        if dev.any():
            data = dev.recv()
            print(f"RECEIVED: id:{hex(data[0])}, ex:{data[1]}, rtr:{data[2]}, data:{data[3]}")
        await asyncio.sleep(0.01)

async def sender():
    counter = 0
    while True:
        # Send a message once per second
        msg_id = 0x123  # CAN message identifier
        # Use list of bytes instead of bytes object
        msg_data = [counter & 0xFF, (counter >> 8) & 0xFF]
        
        # Correct parameter order: data first, then ID
        dev.send(msg_data, msg_id)  # data, id
        
        print(f"SENT: id:{hex(msg_id)}, data:{msg_data}")
        counter += 1
        await asyncio.sleep(1)  # Send message once per second

async def main():
    # Start both tasks concurrently
    read_task = asyncio.create_task(reader())
    send_task = asyncio.create_task(sender())
    
    # Wait for both tasks (this will run forever)
    await asyncio.gather(read_task, send_task)

# Run the example
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
```

### Output Example

```sh
SENT: id:0x123, data:[0, 0]
RECEIVED: id:0x123, ex:False, rtr:False, data:b'\x00\x00'
SENT: id:0x123, data:[1, 0]
RECEIVED: id:0x123, ex:False, rtr:False, data:b'\x01\x00'
SENT: id:0x123, data:[2, 0]
RECEIVED: id:0x123, ex:False, rtr:False, data:b'\x02\x00'
SENT: id:0x123, data:[3, 0]
RECEIVED: id:0x123, ex:False, rtr:False, data:b'\x03\x00'
SENT: id:0x123, data:[4, 0]
RECEIVED: id:0x123, ex:False, rtr:False, data:b'\x04\x00'
```


## Technical Notes

### ESP32 TWAI Timing Compatibility

This module uses a universal timing configuration approach to support all ESP32 variants (ESP32, ESP32-C3, ESP32-S2, ESP32-S3). The original ESP32 has a different `twai_timing_config_t` structure (5 fields) compared to newer variants (7 fields with additional clock source parameters). 

Our solution:
- **ESP32**: Manual timing calculations based on 40MHz APB (Advanced Peripheral Bus) clock frequency
- **ESP32-C3/S2/S3**: Uses ESP-IDF timing macros with conditional compilation  
- **Supported bitrates**: 1k, 5k, 10k, 12.5k, 16k, 20k, 25k, 50k, 100k, 125k, 250k, 500k, 800k, 1000k bps

The timing calculation formula: `Bitrate = APB_CLK_FREQ / (BRP * (1 + tseg_1 + tseg_2))`  
Where APB_CLK_FREQ = 40MHz for ESP32, BRP = Baud Rate Prescaler, tseg_1/tseg_2 = Time segments
