# STM32 UART LED Console Project

## Overview
This project demonstrates a line-based UART command parser for STM32 microcontrollers (e.g., Nucleo boards), integrating interrupt-driven input, echoing, and GPIO control. The system supports human-friendly console editing with backspace handling, real-time command processing, and toggling the onboard LED (PA5) based on textual commands.  

Key features include:
- **Interrupt-driven UART input** using `HAL_UART_Receive_IT` for non-blocking reception.
- **Line-based command processing** (`LED ON`, `LED OFF`) with null-terminated strings.
- **Backspace support**: properly removes characters from the input buffer (`0x08` and `0x7F` handled).
- **Echo functionality**: typed characters are echoed back to the terminal, including edited lines.
- **Command feedback**: each line submission generates a newline and triggers LED control.
- **Simple buffer management**: single `rx_buffer` with `rx_index` tracking current input; optional cyclic buffer can be integrated for high-throughput or continuous streams.
- **Minimal and safe interrupt handling**: no blocking operations inside `HAL_UART_RxCpltCallback`, only storing received bytes and restarting interrupts.
- **UART configuration**: USART2 at 115200 baud, 8-N-1, no flow control; uses ST-LINK Virtual COM Port via mini-USB.

---

## Hardware Setup
- **Board**: STM32 Nucleo-64 (F303/F401/F446)
- **LED**: Onboard PA5 (user LED)
- **UART Connection**: Mini-USB to PC (ST-LINK VCP)
- **Terminal**: PuTTY, minicom, or screen; CR or CR+LF for Enter

---

## Software Implementation

### Interrupt Handler
The `HAL_UART_RxCpltCallback` function:
- Stores each received byte into `rx_buffer` at `rx_index`.
- Handles `\r` (Enter) to terminate the string.
- Handles backspace characters (`\b` and `0x7F`) to remove previous input.
- Immediately restarts the UART receive interrupt for continuous input.

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart){
    if(rx_data == '\r'){
        rx_buffer[rx_index] = '\0'; // terminate string
        if(strcmp((char*)rx_buffer,"LED ON") == 0)
            HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);
        else if(strcmp((char*)rx_buffer,"LED OFF") == 0)
            HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);
        HAL_UART_Transmit(&huart2, (uint8_t*)"\r\n", 2, 100);
        rx_index = 0; // reset buffer
    } else if(rx_data == '\b' || rx_data == 0x7F) {
        if(rx_index > 0) {
            rx_index--;
            rx_buffer[rx_index] = '\0';
        }
    } else {
        if(rx_index < sizeof(rx_buffer)-1){
            rx_buffer[rx_index++] = rx_data;
        }
        HAL_UART_Transmit(&huart2, &rx_data, 1, 100); // echo character
    }
    UNUSED(huart);
    HAL_UART_Receive_IT(&huart2, &rx_data, 1); // restart interrupt
}
