#include <lpc214x.h>
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>

// Define peripherals and pins
#define LCD (0xFFFF00FF)
#define RS (1 << 4)
#define RW (1 << 5)
#define EN (1 << 6)

// Function prototypes
void delay_ms(int count);
void initUART(void);
void LCD_INIT(void);
void LCD_CMD(char command);
void LCD_STRING(char* msg);
void initGPIO(void);
void initRTC(void);
void getTime(int* hour, int* minute);

int main() {
    int count_entry = 0;
    int count_exit = 0;
    int is_daytime = 0;

    int entry_sensor, exit_sensor;
    int hour, minute;

    char buffer[10];

    // Function calls to initialize peripherals
    initUART();
    initGPIO();
    initRTC();
    LCD_INIT(); // Initialize LCD
    while (1) {
        getTime(&hour, &minute);
        if (hour >= 5 && hour < 17) {
            is_daytime = 0; // Daytime
        } else {
            is_daytime = 1; // Nighttime
        }
        entry_sensor = (IO0PIN & (1 << 10)) >> 10;
        exit_sensor = (IO0PIN & (1 << 11)) >> 11;

        if (entry_sensor == 1) {
            count_entry++;
        }

        if (exit_sensor == 1) {
            count_exit++;
        }
        if (count_entry > count_exit && !is_daytime) {
	    IO1SET |= (1 << 20); // Activate LED
	    delay_ms(2000);
        } else {
            IO1CLR |= (1 << 20); // Deactivate LED
        }
	LCD_CMD(0x01);      // Display clear
        sprintf(buffer, "Count: %d", count_entry - count_exit);
        LCD_STRING(buffer);
    }
 return 0;
}
// Function implementations
void delay_ms(int count) {
    int j = 0, i = 0;
    for (j = 0; j < count; j++) {
        // At 60Mhz, the below loop introduces delay of 1 milli sec
        for (i = 0; i < 1250; i++);
    }
}
void initUART(void) {
    PINSEL0 |= 0x00000005; // Enable RxD0 and TxD0
    U0LCR = 0x83;          // 8 bits, no parity, 1 stop bits, DLAB = 1
    U0DLL = 97;            // 9600 Baud Rate @ 15MHZ VPB Clock
    U0LCR = 0x03;          // DLAB = 0
    // Implement UART initialization here
}
void LCD_INIT(void) {
    // P0.12 to P0.15 as output for LCD Data
    // P0.4,5,6 as output for RS, RW and EN
    IO0DIR |= 0x0000F070;
    delay_ms(20);
    LCD_CMD(0x02);  // Initialize cursor to home position
    LCD_CMD(0x28);  // 4 - bit interface, 2 line, 5x8 dots
    LCD_CMD(0x06);  // Auto increment cursor
    LCD_CMD(0x0C);  // Display on cursor off
    LCD_CMD(0x01);  // Display clear
    LCD_CMD(0x80);  // First line first position
}
void LCD_CMD(char command) {
    IO0PIN = ((IO0PIN & LCD) | ((command & 0xF0) << 8)); // Upper nibble
    IO0SET = EN;                                          // EN = 1
    IO0CLR = (RS | RW);                                   // RS = 0, RW = 0
    delay_ms(5);                                          // 5 milli sec
    IO0CLR = EN;                                          // EN = 0, RS and RW unchanged(RS = RW = 0)
    delay_ms(5);
    IO0PIN = ((IO0PIN & LCD) | ((command & 0x0F) << 12)); // Lower nibble
    IO0SET = EN;                                          // EN = 1
    IO0CLR = (RS | RW);                                   // RS = 0, RW = 0
    delay_ms(5);
    IO0CLR = EN;                                          // EN = 0, RS and RW unchanged(RS = RW = 0)
    delay_ms(5);
}
void LCD_STRING(char* msg) {
    unsigned int i = 0;
    while (msg[i] != 0) {
        IO0PIN = ((IO0PIN & LCD) | ((msg[i] & 0xF0) << 8)); // Upper nibble
        IO0SET = (RS | EN);                                  // RS = 1, EN = 1
        IO0CLR = RW;                                         // RW = 0
        delay_ms(2);                                         // 2 milli sec
        IO0CLR = EN;                                         // EN = 0, RS and RW unchanged(RS = 1, RW = 0)
        delay_ms(5);
        IO0PIN = ((IO0PIN & LCD) | ((msg[i] & 0x0F) << 12)); // Lower nibble
        IO0SET = (RS | EN);                                  // RS = 1, EN = 1
        IO0CLR = RW;                                         // RW = 0
        delay_ms(2);
        IO0CLR = EN;                                         // EN = 0, RS and RW unchanged(RS = 1, RW = 0)
        delay_ms(5);
        i++;
    }
}
void initGPIO(void) {
    PINSEL0 = 0x00000000;
    IO0DIR &= ~((1 << 10) | (1 << 11));
    IO1DIR |= (1 << 20);
    // Implement GPIO initialization here
}
void initRTC(void) {
    CCR = 0x11;
    CIIR = 0x01;
    // Implement RTC initialization here
}
void getTime(int* hour, int* minute) {
    *hour = (HOUR & 0x1F);
    *minute = (MIN & 0x3F);
}
