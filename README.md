# Propeller-LED-Display
Propeller POV LED Display using Arduino/8051/LPC2148. Rotating LED array creates text &amp; patterns via persistence of vision. Full code + docs included.
This repository contains the complete Propeller LED Display (POV Display) project.  
A rotating LED bar generates text, numbers, symbols, and animations using the Persistence of Vision principle.

✨ Features
- ATmega328 / Arduino Nano implementation
- 8051 and LPC2148 Embedded C versions
- POV-based rotating LED message display
- Hall-effect sensor / IR sensor synchronization
- 74HC595 / ULN2003 LED driver support

🛠 Hardware Used
- Microcontroller (ATmega328 / Arduino / 8051 / LPC2148)
- 8/16 LED array
- High-speed DC/BLDC motor
- Hall-effect sensor / IR sensor for rotor sync
- LED Driver (ULN2003 / 74HC595)
- 5V regulated power supply
- Lightweight acrylic/PCB propeller blade

📘 What You Can Do
- Display scrolling text
- Create custom animations
- Build a clock-style POV display
- Modify fonts/characters easily

This project is ideal for electronics learners, embedded engineers, and makers exploring POV technologies.

/* LPC2148 conceptual example (ARM7TDMI) — use your SDK/toolchain to adapt
   - Uses external interrupt EINT0 for hall sensor
   - Uses GPIO and a timer for precise delays
   - This is conceptual: adjust register names/peripherals per SDK
*/

// -------------------------------
// Propeller LED Display (POV)
// Arduino Nano / ATmega328
// -------------------------------

#define DATA_PIN 2
#define CLOCK_PIN 3
#define LATCH_PIN 4
#define HALL_PIN 5

volatile bool synced = false;
volatile unsigned long rotation_time = 0;
unsigned long last_time = 0;

const uint8_t font[][8] = {
  {0x3E,0x51,0x49,0x45,0x3E}, // A
  {0x7F,0x49,0x49,0x49,0x36}, // B
  {0x3E,0x41,0x41,0x41,0x22}, // C
};

void IRAM_ATTR hall_ISR() {
  unsigned long now = micros();
  rotation_time = now - last_time;
  last_time = now;
  synced = true;
}

void sendData(uint8_t data) {
  digitalWrite(LATCH_PIN, LOW);
  shiftOut(DATA_PIN, CLOCK_PIN, MSBFIRST, data);
  digitalWrite(LATCH_PIN, HIGH);
}

void displayChar(const uint8_t *pattern) {
  int columns = 5;
  unsigned long delay_time = rotation_time / (columns + 5);

  for (int i = 0; i < columns; i++) {
    sendData(pattern[i]);
    delayMicroseconds(delay_time);
  }

  sendData(0x00);
  delayMicroseconds(delay_time * 2);
}

void setup() {
  pinMode(DATA_PIN, OUTPUT);
  pinMode(CLOCK_PIN, OUTPUT);
  pinMode(LATCH_PIN, OUTPUT);
  pinMode(HALL_PIN, INPUT_PULLUP);

  attachInterrupt(digitalPinToInterrupt(HALL_PIN), hall_ISR, FALLING);
}

void loop() {
  if (!synced) return;

  displayChar(font[0]);  // A
  displayChar(font[1]);  // B
  displayChar(font[2]);  // C
}
// ---------------------------------
// Propeller LED Display – 8051 Code
// ---------------------------------

#include <REGX51.H>

#define LED P2
sbit HALL = P3^2;

unsigned int rotation_time = 5000;

// 5x8 Font for A, B, C
unsigned char font[3][5] = {
  {0x3E,0x51,0x49,0x45,0x3E},
  {0x7F,0x49,0x49,0x49,0x36},
  {0x3E,0x41,0x41,0x41,0x22}
};

void delay_us(unsigned int t) {
  unsigned int i;
  for (i=0;i<t;i++);
}

void hall_ISR(void) interrupt 0 {
  rotation_time = 5000; // simpler static timing
}

void displayChar(unsigned char *pat) {
  unsigned char i;
  for (i=0;i<5;i++) {
    LED = pat[i];
    delay_us(rotation_time);
  }
  LED = 0x00;
  delay_us(rotation_time);
}

void main() {
  IT0 = 1;
  EX0 = 1;
  EA = 1;

  while(1) {
    displayChar(font[0]);
    displayChar(font[1]);
    displayChar(font[2]);
  }
}
// -----------------------------------------
// Propeller LED Display – LPC2148 ARM7
// -----------------------------------------

#include <lpc214x.h>

volatile unsigned long rotation_time = 3000;

const unsigned char font[3][5] = {
  {0x3E,0x51,0x49,0x45,0x3E},
  {0x7F,0x49,0x49,0x49,0x36},
  {0x3E,0x41,0x41,0x41,0x22}
};

void delay_us(unsigned int t) {
  unsigned int i;
  for(i=0; i<t*10; i++);
}

void EINT0_Handler(void) __irq {
  rotation_time = 3000;
  EXTINT = 1;
  VICVectAddr = 0;
}

void displayChar(const unsigned char *p) {
  int i;
  for (i=0; i<5; i++) {
    IO0SET = p[i];
    delay_us(rotation_time);
    IO0CLR = 0xFF;
  }
}

int main(void) {
  PINSEL0 |= 1<<0;   // EINT0
  VICVectAddr0 = (unsigned long)EINT0_Handler;
  VICVectCntl0 = 0x20 | 14;
  VICIntEnable = 1<<14;

  IO0DIR = 0xFFFFFFFF;

  while(1) {
    displayChar(font[0]);
    displayChar(font[1]);
    displayChar(font[2]);
  }
}
