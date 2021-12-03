# Lab6_Part2
#include <msp430.h>

volatile long tempRaw , temp;  //volatile -> the object can be modified, long -> gives a greater range of what can be stored
char result[100]; // stores 100 temperature values
volatile long sample[100];// sample type

volatile long humRaw;// Raw data from humidity sensor
char humResult[100];   // This will hold the converted raw data
volatile long Hsample[100]; // Stores each data point


void ConfigureAdc_temp();
void ConfigureADC_Humidity(); // Header for the function that will configure the ADC

void uart_init(void);
void ConfigClocks(void);
void strreverse(char* begin, char* end);
void itoa(int value, char* str, int base);
void port_init();

void main(void)
{
    WDTCTL = WDTPW + WDTHOLD; // Stop watchdog timer


    port_init();
    ConfigClocks();
    uart_init();

    _delay_cycles(5);                // Wait for ADC Ref to settle

    while(1){

//Sensor 1:
        int flip = 0;
        while (flip == 0)
        {
                ConfigureAdc_temp();
                ADC10CTL0 |= ENC + ADC10SC +MSC;       // Converter Enable, Sampling/conversion start
                while((ADC10CTL0 & ADC10IFG) == 0);    // check the Flag, while its low just wait
                P1OUT |= BIT0;                         // green LED on
               // _delay_cycles(20000000);               // delay for about 1 second so LED doesn't flash too fast
                tempRaw = ADC10MEM;                    // read the converted data into a variable
                temp = (tempRaw* 48724 - 30634388) >> 16;
                P1OUT &= ~BIT0;                        // green LED off
                ADC10CTL0 &= ~ADC10IFG;                // clear the flag
                itoa(((tempRaw* 48724 - 30634388) >> 16),result,10); // convert temperature reading to Fahrenheit

                int acount =0;

                                    while(result[acount]!='\0')            // goes through the string until it gets to the end
                                  {
                                      while((IFG2 & UCA0TXIFG)==0);                     //Wait Until the UART transmitter is ready
                                      UCA0TXBUF = result[acount++] ;                   //Transmit the received data.

                                  }
                flip++;
        }
        while (flip == 1)
        {
//Sesnor 2:
                ConfigureADC_Humidity();
                ADC10CTL0 |= ENC + ADC10SC + MSC;      // Enable Converter, Start multiple sample/conversion, Take multiple samples automatically
                while((ADC10CTL0 & ADC10IFG) == 0);    // checks the Flag, while its low just wait
                P1OUT |= BIT6;                         // turns the RED led on, to show that this path was taken
               // _delay_cycles(20000000);               // delays the LED turning off so we can catch it turning on
                humRaw = ADC10MEM;                     // reads the converted analog data into a variable
                P1OUT &= ~BIT6;                        // Turns the red LED off
                ADC10CTL0 &= ~ADC10IFG;                // Resets flag
                itoa(humRaw, humResult, 10);           // converts the data type in humRaw into a string data type and store it in humResult, with number system being base 10

                int bcount = 0;
                while(result[bcount]!='\0')
                {
                    while((IFG2 & UCA0TXIFG) == 0);    // Waits until the UART transmitter is ready
                    UCA0TXBUF = humResult[bcount++];    // Transmit the received data
                }
                flip--;
        }


    }
}

// Configure ADC for Humidity Sensor
void ConfigureADC_Humidity()
{
    ADC10CTL1 = INCH_13 + ADC10DIV_0 + CONSEQ_2;        //INCH Selects the correct pin, DIV_0 divides clock by 1, CONSEQ_2 multiple sample/conversion operations out of single channel
    ADC10CTL0 = ADC10SHT_3 | ADC10ON;                   //Turns the ADC on, 64 ATD clocks per sample, no references

}


// Configure ADC Temperature
void ConfigureAdc_temp()
{
     ADC10CTL1 = INCH_10 + ADC10DIV_0 + CONSEQ_2;
     ADC10CTL0 = SREF_1 | ADC10SHT_3 | REFON | ADC10ON ;//| ADC10IE; //Vref+, Vss, 64 ATD clocks per sample, internal references, turn ADCON
     __delay_cycles(5);                                 //wait for adc Ref to settle
      ADC10CTL0 |= ENC| MSC;                            //converter Enable, Sampling/Conversion start, multiple sample/conversion operations
}


void uart_init(void){
    UCA0CTL1 |= UCSWRST;                     //Disable the UART state machine
    UCA0CTL1 |= UCSSEL_3;                    //Select SMCLK as the baud rate generator source
    UCA0BR1 =0;
    UCA0BR0 = 104;                           //Produce a 9,600 Baud UART rate
    UCA0MCTL = 0x02;                         //Choose appropriately from Table 15-4 in User Guide
    UCA0CTL1 &= ~UCSWRST;                    //Enable the UART state naching
    IE2 |= UCA0RXIE;                         //Enable the UART receiver Interrupt
}
void ConfigClocks(void)
 {

  BCSCTL1 = CALBC1_1MHZ;                     // Set range
  DCOCTL = CALDCO_1MHZ;                      // Set DCO step + modulation
  BCSCTL3 |= LFXT1S_2;                       // LFXT1 = VLO
  IFG1 &= ~OFIFG;                            // Clear OSCFault flag
  BCSCTL2 = 0;                               // MCLK = DCO = SMCLK
 }



void strreverse(char* begin, char* end)      // Function to reverse the order of the ASCII char array elements
{
    char aux;
    while(end>begin)
        aux=*end, *end--=*begin, *begin++=aux;
}

void itoa(int value, char* str, int base) //Function to convert the signed int to an ASCII char array
{

    static char num[] = "0123456789abcdefghijklmnopqrstuvwxyz";
    char* wstr=str;
    int sign;

    // Validate that base is between 2 and 35 (inlcusive)
    if (base<2 || base>35)
    {
        *wstr='\0';
        return;
    }

    // Get magnitude and th value
    sign=value;
    if (sign < 0)
        value = -value;

    do                                      // Perform integer-to-string conversion.
        *wstr++ = num[value%base];          //create the next number in converse by taking the modolus
    while(value/=base);                     // stop when you get a 0 for the quotient

    if(sign<0) //attach sign character, if needed
        *wstr++='-';
    *wstr='\0'; //Attach a null character at end of char array. The string is in revers order at this point
    strreverse(str,wstr-1); // Reverse string

}

void port_init()
{
    P1SEL |= BIT1 + BIT2;            // select non-GPIO  usage for Pins 1 and 2
    P1SEL2 |= BIT1 + BIT2;           // Select UART usage of Pins 1 and 2
}

