/*
 *    www.SAMcontrollers.com
 *      DC motor driver DEMO02 project - SAM-VNH7DC12A  https://samcontrollers.com/product/dc-12a-h-bridge-driver-board/
 *      Control two LED Green and Orange showing polarity direction PWM change brightness by variable pot
 *      Or control DC motor connected to A and B outputs of SAM-VNH7DC12A
 *      Use heatsink for higher current loads to prevent damage / overheating
 *      
 * Arduino NANO or DUE  or other (change pinout accordingly)  
 * 
 *   Pin used and connected to:
 *     NANO/DUE      SAM-VNH7DC12A
 *      D02 Output - IN_A + SEL_0
 *      D03 Output - IN_B
 *      D05 Output - PWM
 *      D11 Output - Blue LED 
 *      
 *      A0 - Input Resistor Pot input
 *      A6 - CS_OUT
 *      
 */
#include <LiquidCrystal_I2C.h>

//#define SerialPrint
//#define Display_A           // Display are "A" type

uint16_t Speed = 0 ;                              // Speed variable
char * Status = "     ";                          // Output status: STOP, Left, Right
byte StatusFlag = 0;                              // Ststus Flag - 0:STOP, 1:Left, 2:Right

const int numReadings = 25;       // Quantity of readings
float average = 0;                //  Averaging result
float koefI = 1.2;                // Current calc coeficient

//                    addr, en,rw,rs,d4,d5,d6,d7,bl,blpol
#if defined (Display_A)
LiquidCrystal_I2C lcd(0x3F, 2, 1, 0, 4, 5, 6, 7, 3, POSITIVE);  //  address 0x3F PCF8574AT  MARKED  "A" on the display front
#else 
LiquidCrystal_I2C lcd(0x27, 2, 1, 0, 4, 5, 6, 7, 3, POSITIVE);  //  address 0x27 PCF8574T  NO MARK on the display front
#endif


// Resistor output recalculation 
uint16_t ZeroWidth = 40;
uint16_t LeftMin = 0;
uint16_t LeftMax = ((1023/2)-(ZeroWidth/2));
uint16_t RightMin = LeftMax + ZeroWidth;
uint16_t RightMax = 1023;

/////////////////////////////////////////////////  SETUP  ///////////////////////////
void setup(){
#if defined (SerialPrint)
  Serial.begin(9600);   // Serial communication setup
  Serial.print(F(" DC Mptor Driver Test"));
#endif

    lcd.begin(20, 4);  lcd.clear();  lcd.noBacklight();  lcd.backlight();     // Seting up a 2004 display size
    
    pinMode(2, OUTPUT);         // D02 Output - IN_A
    digitalWrite(2, LOW);         // IN_A
    pinMode(3, OUTPUT);         // D03 Output - IN_B
    digitalWrite(3, LOW);         // IN_B
//    pinMode(4, OUTPUT);         // D04 Output - SEL_0    connect SEL_0 with IN_A
//    digitalWrite(4, LOW);
    pinMode(5, OUTPUT);         // D05 Output - PWM
    digitalWrite(5, LOW);
//                                   01234567890123456789
    lcd.setCursor(0, 0);  lcd.print("demo02 SAM-VNH7DC12A");
    lcd.setCursor(0, 1);  lcd.print("    Dir: ");
    lcd.setCursor(0, 2);  lcd.print("  Speed: ");
    lcd.setCursor(0, 3);  lcd.print("Current: ");
}

/////////////////////////////////////////////////  MAIN  ///////////////////////////
void loop() {  
        Speed = analogRead(A0); delay(1); Speed = analogRead(A0);         // Read Pot voltage
        if (Speed < LeftMax ) {
//              map(value, fromLow, fromHigh, toLow, toHigh)
          Speed = map(Speed, LeftMax, LeftMin, 0, 254);   // Convert to 0..254 range
          digitalWrite(2, HIGH);                        // IN_A
          digitalWrite(3, LOW);                         // IN_B
//          digitalWrite(4, HIGH);                      // SEL_0 - Current Monitoring ON
          StatusFlag = 1;                               // Ststus Flag - 0:STOP, 1:Left, 2:Right
          Status = "Left ";                             // Output status
        } else if (Speed > RightMin ) {
          Speed = map(Speed, RightMin, RightMax, 0, 254);   // Convert to 0..254 range
          digitalWrite(2, LOW);                         // IN_A
          digitalWrite(3, HIGH);                        // IN_B
//          digitalWrite(4, LOW);                       // SEL_0 - Current Monitoring ON
          StatusFlag = 2;                               // Ststus Flag - 0:STOP, 1:Left, 2:Right
          Status = "Right";                             // Output status
        } else {
          Speed = 0;
          digitalWrite(2, LOW);                         // IN_A
          digitalWrite(3, LOW);                         // IN_B
//          digitalWrite(4, LOW);                       // SEL_0 - Current Monitoring ON
          StatusFlag = 0;                               // Ststus Flag - 0:STOP, 1:Left, 2:Right
          Status = "STOP ";                             // Output status
        }     
        analogWrite(5, Speed);                          // PWM Output
        analogWrite(11, Speed);                         // LED PWM Output      
//              map(value, fromLow, fromHigh, toLow, toHigh)
        Speed = map(Speed, 0, 254, 0, 100);             // Convert to %% of power output
#if defined (SerialPrint)
        Serial.print("-> "); Serial.print(Status);
        Serial.print("  Speed: "); Serial.print(Speed); Serial.print("%");
#endif
//                          01234567890123456789
    lcd.setCursor(9, 1);  lcd.print("           ");
    lcd.setCursor(9, 2);  lcd.print("           ");
    lcd.setCursor(9, 1);  lcd.print(Status);
    lcd.setCursor(9, 2);  lcd.print(Speed);  lcd.print("%");
    lcd.setCursor(9, 3);  lcd.print("           ");
        switch (StatusFlag) {
          case 0:                       // 0:STOP
            break;
          case 1:                       // 1:Left
            MeasurementSensInput();
#if defined (SerialPrint)
            Serial.print("  Current A: "); Serial.print(float (average)); Serial.print("mA");
#endif
            lcd.setCursor(9, 3); lcd.print(float (average)); lcd.print("mA");
            break;
          case 2:                       // 2:Right
            MeasurementSensInput();
#if defined (SerialPrint)
            Serial.print("  Current B: "); Serial.print(float (average)); Serial.print("mA");
#endif
            lcd.setCursor(9, 3); lcd.print(float (average)); lcd.print("mA");
            break;
        }    
#if defined (SerialPrint)
        Serial.println();
#endif
        delay (150);
}

/////////////////////////////////////////////////  MeasurementSensInput  ///////////////////////////
void MeasurementSensInput() {     // Acquire Current Sens and averaging
  average = analogRead(6);
  for (int i = 0; i < numReadings; i++) {
    average = average + analogRead(6); delay(1);
  }
  average = average/numReadings;
  average = average * 4 * koefI;
  delay(1);
}
