# Sleep-Mode-in-Energia
Energia program for the MSP430G2XX to demonstrate toggling in and out of lowest-power sleep mode.


// Energia program for the MSP430G2XX to demonstrate toggling in and out of lowest-power sleep mode.
// V. Goncharoff, University of Illinois at Chicago, Spring 2015 Semester.
// volodia@uic.edu, www.goncharoff.homestead.com
// (FOR UNLIMITED PUBLIC DISTRIBUTION)
//
// This program demonstrates how one may avoid using a power switch by instead putting the MSP430G2XX
// microcontroller to sleep using a function call. The microcontroller goes to sleep (in Low Power Mode 4,
// the lowest-power-consuming sleep state) when switch S2 is pressed, and wakes up when S2 is pressed again.
//
// If you are using the TI LaunchPad, you may add a momentary pushbutton switch between pin 6 and GND
// to change the green LED's brightness.  Otherwise, S2 and the green LED that are already installed on the
// LaunchPad development board are enough to demonstrate the sleep/wake function.


// Define constants and declare global variables
const int LED = 14;                 // pin 14: green LED on the LaunchPad board
const int delay_value = 5;          // controls the speed with which the LED fades in and out
const int fade_switch = 6;          // normally-open momentary contact switch from pin 6 to GND
const int sleep_switch = 5;         // normally-open momentary contact switch from pin 5 to GND (S2 on LaunchPad)
const int debounce_delay = 250;     // 1/4 sec debouncing delay after sleep switch is closed or opened
const int fadeValue_min = 50;       // sets the lowest LED brightness value
const int fadeValue_max = 255;      // sets the highest LED brightness value
const int fadeValue_startup = 128;  // sets the initial LED brightness value
int fadeValue_step = 1;             // this value gets negated when endpoints are reached (not a constant)
int fadeValue = 0;                  // define global variable that is used to specify LED brightness


//---------------------------------------------------------------------------------------

void setup() {
  Initialize_Pins();   // configure I/O pins for normal operation
  fadeValue = fadeValue_startup;
}

//---------------------------------------------------------------------------------------

void loop() {

  Output_to_LED(fadeValue,LED);
    
  while (digitalRead(fade_switch) == LOW) {
    fadeValue = min(max(fadeValue+fadeValue_step,fadeValue_min),fadeValue_max);
    Output_to_LED(fadeValue,LED);
    delay(delay_value);
    if ((fadeValue == fadeValue_max) | (fadeValue == fadeValue_min)) {
      fadeValue_step = -fadeValue_step;
    }
  }
  
  if (digitalRead(sleep_switch) == LOW) {
      Fall_Asleep();
  }
  
}
//---------------------------------------------------------------------------------------

void Output_to_LED(int level, int LED_pin) {
// Function to send PWM signal to LED_pin (0 <= level <= 255)
    // nonlinearly process level to linearize the perceived LED brightness:
    float ftemp = float(level)/255;
    int   itemp = int(255*ftemp*ftemp*ftemp);
    // send PWM code to the specified microcontroller output pin:
    analogWrite(LED_pin,itemp);
}

//---------------------------------------------------------------------------------------

void Initialize_Pins(void) {
  // Function to intialize direction of the I/O pins
  // (do not remove, as this function is called by Fall_Asleep)
  pinMode(LED,OUTPUT);
  pinMode(fade_switch,INPUT_PULLUP);
  pinMode(sleep_switch,INPUT_PULLUP);
}


//---------------------------------------------------------------------------------------
// Change nothing below (is needed for going to sleep and waking up again)
// --------------------------------------------------------------------------------------

// Energia function for the MSP430G2XX to put it into lowest-power sleep mode and wake up with S2.
// V. Goncharoff, University of Illinois at Chicago, Spring 2015 Semester.
// volodia@uic.edu, www.goncharoff.homestead.com
// (FOR UNLIMITED PUBLIC DISTRIBUTION)
//
// Call this function to put uC into deep sleep.  HIGH to LOW transition at pin 5 (switch S2 on
// LaunchPad) then wakes it up and returns to the calling procedure in full active mode.

void Fall_Asleep(void) {
  delay(debounce_delay);     // delay for approximately 250ms, longer than duration of switch bounce
  // Put all I/O pins into input mode to save power when uC goes to sleep:
  pinMode(2,INPUT);
  pinMode(3,INPUT);
  pinMode(4,INPUT);
  pinMode(5,INPUT_PULLUP);   // pullup resistor for switch that will interrupt sleep mode
  pinMode(6,INPUT);
  pinMode(7,INPUT);
  pinMode(8,INPUT);
  pinMode(9,INPUT);
  pinMode(10,INPUT);
  pinMode(11,INPUT);
  pinMode(12,INPUT);
  pinMode(13,INPUT);
  pinMode(14,INPUT);
  pinMode(15,INPUT);
  // Enable interrupt upon HIGH to LOW transition at pin 5 (P1.3, S2 on LaunchPad board)
  P1IE |= BIT3;              // Enable interrupt by change in voltage at pin P1.3
  P1IES |= BIT3;             // Specifically, trigger interrupt at HIGH to LOW transition in P1.3
  P1IFG &= ~BIT3;            // Clear interrupt flage IFG corresponding to pin P1.3
  // fall asleep:
  _BIS_SR(LPM4_bits + GIE);  // Enter Low Power Mode 4 w/interrupt
  
  // upon waking up:
  delay(debounce_delay);           // delay for approximately 250ms, longer than duration of switch bounce
  Initialize_Pins();               // Initialize I/O pins after sleep mode
  Output_to_LED(fadeValue,LED);    // Light the LED at the previous brightness level
  // wait for user to release the wake-up switch:
  while (digitalRead(sleep_switch) == LOW) {
  }
}

// Interrupt service routine that executes when pin 5 transitions from HIGH to LOW:
#pragma vector=PORT1_VECTOR
__interrupt void Port_1(void)  {
  __bic_SR_register_on_exit(LPM4_bits+GIE); // Resume execution in Active Mode
}
