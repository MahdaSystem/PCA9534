# PCA9534 Library
Driver for PCA9534 and PCA9534A 8-bit I/O expander.


## Hardware Support
It is easy to port this library to any platform. But now it is ready for use in:
- ESP32 (esp-idf)


## How To Use
1. Add `PCA9534.h` and `PCA9534.c` files to your project.  It is optional to use `PCA9534_platform.h` and `PCA9534_platform.c` files (open and config `PCA9534_platform.h` file).
2. Initialize platform-dependent part of handler.
4. Call `PCA9534_Init()`.
5. Call other functions and enjoy.


## Example
<details>
<summary>Using PCA9534_platform files</summary>

```c
#include <stdio.h>
#include "PCA9534.h"
#include "PCA9534_platform.h"

int main(void)
{
  PCA9534_Handler_t Handler = {0};
  uint8_t OutData = 0x00;
  uint8_t InData = 0x00;

  PCA9534_Platform_Init(&Handler);
  PCA9534_Init(&Handler, PCA9534_DEVICE_PCA9534A, 0);
  PCA9534_SetDir(&Handler, 0x01);

  while (1)
  {
    OutData ^= 0x01;
    PCA9534_Write(&Handler, OutData);

    PCA9534_Read(&Handler, &InData);
    printf("Read Data: 0x%02X\r\n\r\n",
           InData);
  }

  PCA9534_DeInit(&Handler);
  return 0;
}
```
</details>


<details>
<summary>Without using PCA9534_platform files (esp-idf)</summary>

```c
#include <stdio.h>
#include <stdint.h>
#include "sdkconfig.h"
#include "esp_system.h"
#include "esp_err.h"
#include "driver/i2c.h"
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "PCA9534.h"

#define PCA9534_I2C_NUM   I2C_NUM_1
#define PCA9534_I2C_RATE  100000
#define PCA9534_SCL_GPIO  GPIO_NUM_13
#define PCA9534_SDA_GPIO  GPIO_NUM_14

int8_t
PCA9534_Platform_Init(void)
{
  i2c_config_t conf = {0};
  conf.mode = I2C_MODE_MASTER;
  conf.sda_io_num = PCA9534_SDA_GPIO;
  conf.sda_pullup_en = GPIO_PULLUP_DISABLE;
  conf.scl_io_num = PCA9534_SCL_GPIO;
  conf.scl_pullup_en = GPIO_PULLUP_DISABLE;
  conf.master.clk_speed = PCA9534_I2C_RATE;
  if (i2c_param_config(PCA9534_I2C_NUM, &conf) != ESP_OK)
    return -1;
  if (i2c_driver_install(PCA9534_I2C_NUM, conf.mode, 0, 0, 0) != ESP_OK)
    return -1;
  return 0;
}

int8_t
PCA9534_Platform_DeInit(void)
{
  i2c_driver_delete(PCA9534_I2C_NUM);
  gpio_reset_pin(PCA9534_SDA_GPIO);
  gpio_reset_pin(PCA9534_SCL_GPIO);
  return 0;
}

int8_t
PCA9534_Platform_Send(uint8_t Address, uint8_t *Data, uint8_t DataLen)
{
  i2c_cmd_handle_t PCA9534_i2c_cmd_handle = {0};
  Address <<= 1;
  Address &= 0xFE;

  PCA9534_i2c_cmd_handle = i2c_cmd_link_create();
  i2c_master_start(PCA9534_i2c_cmd_handle);
  i2c_master_write(PCA9534_i2c_cmd_handle, &Address, 1, 1);
  i2c_master_write(PCA9534_i2c_cmd_handle, Data, DataLen, 1);
  i2c_master_stop(PCA9534_i2c_cmd_handle);
  if (i2c_master_cmd_begin(PCA9534_I2C_NUM, PCA9534_i2c_cmd_handle, 1000 / portTICK_PERIOD_MS) != ESP_OK)
  {
    i2c_cmd_link_delete(PCA9534_i2c_cmd_handle);
    return -1;
  }
  i2c_cmd_link_delete(PCA9534_i2c_cmd_handle);
  return 0;
}

int8_t
PCA9534_Platform_Receive(uint8_t Address, uint8_t *Data, uint8_t DataLen)
{
  i2c_cmd_handle_t PCA9534_i2c_cmd_handle = {0};
  Address <<= 1;
  Address |= 0x01;

  PCA9534_i2c_cmd_handle = i2c_cmd_link_create();
  i2c_master_start(PCA9534_i2c_cmd_handle);
  i2c_master_write(PCA9534_i2c_cmd_handle, &Address, 1, 1);
  i2c_master_read(PCA9534_i2c_cmd_handle, Data, DataLen, I2C_MASTER_LAST_NACK);
  i2c_master_stop(PCA9534_i2c_cmd_handle);
  if (i2c_master_cmd_begin(PCA9534_I2C_NUM, PCA9534_i2c_cmd_handle, 1000 / portTICK_PERIOD_MS) != ESP_OK)
  {
    i2c_cmd_link_delete(PCA9534_i2c_cmd_handle);
    return -1;
  }
  i2c_cmd_link_delete(PCA9534_i2c_cmd_handle);
  return 0;
}


void app_main(void)
{
  PCA9534_Handler_t Handler = {0};
  uint8_t OutData = 0x00;
  uint8_t InData = 0x00;

  PCA9534_PLATFORM_LINK_INIT(Handler, PCA9534_Platform_Init);
  PCA9534_PLATFORM_LINK_DEINIT(Handler, PCA9534_Platform_DeInit);
  PCA9534_PLATFORM_LINK_SEND(Handler, PCA9534_Platform_Send);
  PCA9534_PLATFORM_LINK_RECEIVE(Handler, PCA9534_Platform_Receive);

  PCA9534_Init(&Handler, PCA9534_DEVICE_PCA9534A, 0);
  PCA9534_SetDir(&Handler, 0x01);

  while (1)
  {
    OutData ^= 0x01;
    PCA9534_Write(&Handler, OutData);

    PCA9534_Read(&Handler, &InData);
    printf("Read Data: 0x%02X\r\n\r\n",
           InData);

    vTaskDelay(1000 / portTICK_PERIOD_MS);
  }

  PCA9534_DeInit(&Handler);
}
```
</details>