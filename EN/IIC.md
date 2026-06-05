# Table of Contents
1. Physical Bus
2. Communication Timing
3. Supplementary Notes and Summary
<br><br><br>






# 1. Physical Bus
The IIC bus has two lines:
<br>(1) Clock line SCL
<br>(2) Data line SDA
<br>Multiple slave devices are connected in parallel on the bus. For SCL, in most applications, the slave does not actively control SCL, but the IIC specification allows the slave to pull SCL low to make the master wait. This article does not discuss that; thus, the slave is always in a high-impedance state, and only the master provides a synchronization signal to each slave by sending low-level pulses. For SDA, the transmitting side can either pull the line low or release it to a high level through high impedance, while the receiving side maintains a high-impedance state to read the level. Therefore, SCL carries only a unidirectional signal from the master, while SDA carries a bidirectional signal.
<br>Both SCL and SDA are open-drain structures. It is recommended to add pull-up resistors on both lines to stabilize the high level when the pins are released, and to limit the current flowing into the chip with an appropriate resistance value. (In typical 3.3V-5V applications, a 4.7kΩ resistor is recommended.)
<br>![IIC_Bus](https://github.com/INKPTR-STUDIO/oled-display-ssd1306/blob/main/Images/IIC_Bus.png)
<br><br><br>






# 2. Communication Timing
In terms of level logic, both master and slave devices sample SDA when SCL is high, and then interpret it as data or commands depending on the situation:
- When SCL is high, if SDA remains unchanged, that is considered data transmission. SDA being high means a 1 is transmitted, and SDA being low means a 0 is transmitted.
- When SCL is high, a level transition on SDA is considered a start or stop command. A falling edge on SDA is a start command, and a rising edge on SDA is a stop command. (In fact, these start and stop commands are exactly the start and stop signal timings described below.)

These level logic patterns combine into six types of timing: Start Signal, Stop Signal, Send Acknowledge, Receive Acknowledge, Send Byte, Receive Byte. By combining the required timings according to the device's communication requirements, IIC communication can be achieved. The timing details are explained below.
<br>![IIC_Timing](https://github.com/INKPTR-STUDIO/oled-display-ssd1306/blob/main/Images/IIC_Timing.png)
<br>



## 2.1 Start Signal
*Tells the device that communication is starting.*
<br>
<br>Logic requirement: When SCL is **high**, a **falling edge** must occur on SDA.

|Step|Level Operation|Purpose|
|:-:|:-:|:-|
|1|SDA = 1|Prepare SDA for generating a falling edge|
|2|SCL = 1|Prepare to detect a falling edge on SDA while SCL is high|
|3|SDA = 0|Generate a falling edge on SDA|
|4|SCL = 0|Bring SCL into the working state|

Code reference:
```C++
void IIC_Start(void) {
    SDA = 1;
    SCL = 1;
    SDA = 0;
    SCL = 0;
}
```
<br>



## 2.2 Stop Signal
*Tells the device that the current communication has ended. After power-on, it is recommended to send a stop signal first to prevent the slave from malfunctioning due to an uncertain level state, which could cause an abnormal first communication, and to initialize the bus levels to a known state.*
<br>
<br>Logic requirement: When SCL is **high**, a **rising edge** must occur on SDA.

|Step|Level Operation|Purpose|
|:-:|:-:|:-|
|1|SCL = 0|Prevent SDA from being falsely triggered while SCL is high|
|2|SDA = 0|Prepare SDA for generating a rising edge|
|3|SCL = 1|Prepare to detect a rising edge on SDA|
|4|SDA = 1|Generate a rising edge on SDA|

Code reference:
```C++
void IIC_Stop(void) {
    SCL = 0;
    SDA = 0;
    SCL = 1;
    SDA = 1;
}
```
<br>



## 2.3 Send Acknowledge
*Typically used after receiving a byte, and can be used to control the slave's subsequent operation. (For example, when reading data from a memory chip, sending an acknowledge after receiving a byte usually indicates that the master will continue to receive the next byte from the memory chip; sending a not-acknowledge indicates that no more bytes will be received. Different chips may respond to acknowledge signals in different ways; refer to the data sheet of the chip you are using for details.)*
<br>
<br>Logic requirement: After preparing the acknowledge level on SDA, generate a **high-level pulse** on SCL.
<br>Acknowledge: ACK = 0 is considered an acknowledge, otherwise not acknowledge; same below.

|Step|Level Operation|Purpose|
|:-:|:-:|:-|
|1|SDA = ACK|Prepare the acknowledge level on SDA|
|2|SCL = 1|Generate a high-level pulse on SCL|
|3|SCL = 0|End the high-level pulse on SCL|

Code reference:
```C++
void IIC_SendACK(u8 ACK) {    // <-- u8 stands for unsigned char, same below
    if(ACK) {SDA = 1;}  // <-- Written this way because it uses the truth value of ACK, i.e., only whether ACK == 0 or ACK != 0. A direct SDA = ACK assignment may not be suitable or could cause abnormal pin register operation, especially when ACK > 1. Same below.
    else {SDA = 0;}
    SCL = 1;
    SCL = 0;
}
```
<br>



## 2.4 Receive Acknowledge
*Typically used after sending a byte, and can be used to determine whether the slave has completed its internal operation and is ready to continue communication. (For example, when writing data to a memory chip, after the master sends data, the chip needs time to control the memory cells to charge/discharge and store the data. During this process, the chip generally does not acknowledge; it sends an acknowledge after the operation is complete, allowing the communication to continue. Different chips may have different acknowledge signals; refer to the data sheet of the chip you are using for details.)*
<br>
<br>Logic requirement: First ensure SDA is in a readable state; **when SCL is high, read the level of SDA**, then restore SCL to low.

|Step|Level Operation|Purpose|
|:-:|:-:|:-|
|1|SDA = 1|Release the bus, making SDA readable|
|2|SCL = 1|Inform the slave that the master is sampling SDA|
|3|ACK = SDA|Sample the SDA level as ACK|
|4|SCL = 0|Inform the slave that the master has finished sampling the SDA level|

Code reference:
```C++
u8 IIC_ReceiveACK(void) {
    u8 ACK;

    SDA = 1;
    SCL = 1;
    ACK = SDA;
    SCL = 0;
    return ACK;
}
```
<br>



## 2.5 Send Byte
Logic requirement: From most significant to least, prepare each bit of the byte to be sent on SDA, then generate a high-level pulse on SCL, repeating 8 times (i.e., D7, D6, D5, ..., D0).
<br>Example: Suppose the byte to be sent is 0xCA, whose binary is 1100 1010. Then each time SCL generates a high-level pulse, the SDA levels will be 1, 1, 0, 0, 1, 0, 1, 0 in sequence.

|Step|Level Operation|Purpose|
|:-:|:-:|:-|
|1|SDA = D7|Prepare the level of the D7 bit of the byte on SDA|
|2|SCL = 1|Generate a high-level pulse on SCL|
|3|SCL = 0|End the high-level pulse on SCL|
|4|Repeat steps 1-3, changing SDA = D6|Send the D6 bit of the byte|
|5|Repeat steps 1-3, changing SDA = D5|Send the D5 bit of the byte|
|...|...|...|
|10|Repeat steps 1-3, changing SDA = D0|Send the D0 bit of the byte|

Code reference:
```C++
void IIC_SendByte(u8 Byte) {
    u8 i;

    for(i = 0 ; i < 8 ; i++) {
        if(Byte & 0x80) {SDA = 1;}
        else {SDA = 0;}
        SCL = 1;
        SCL = 0;
        Byte = Byte << 1;
    }
}
```
<br>



## 2.6 Receive Byte
Logic requirement: First ensure SDA is in a readable state; when SCL is high, read the SDA level, then restore SCL to low. The read levels are taken in sequence as the bits of the byte from most significant to least, repeating 8 times to obtain the received byte.

|Step|Level Operation|Purpose|
|:-:|:-:|:-|
|1|SDA = 1|Release the bus, making SDA readable|
|2|SCL = 1|Inform the slave that the master is sampling SDA|
|3|D7 = SDA|Sample the SDA level as the D7 bit of the received byte|
|4|SCL = 0|Inform the slave that the master has finished sampling the SDA level|
|5|Repeat steps 2-4, changing D6 = SDA|Receive the D6 bit of the byte|
|6|Repeat steps 2-4, changing D5 = SDA|Receive the D5 bit of the byte|
|...|...|...|
|11|Repeat steps 2-4, changing D0 = SDA|Receive the D0 bit of the byte|

Code reference:
```C++
u8 IIC_ReceiveByte(void) {
    u8 i, Byte=0x00;

    SDA = 1;
    for(i = 0 ; i < 8 ; i++) {
        Byte = Byte << 1;
        SCL = 1;
        if(SDA) {Byte |= 0x01;}
        SCL = 0;
    }
    return Byte;
}
```
<br><br><br>






# 3. Supplementary Notes and Summary
## 3.1 Explanation of the Initial State of SCL in Some Timings
In the above examples, the four timings—Send Acknowledge, Receive Acknowledge, Send Byte, and Receive Byte—do not initialize SCL to 0 at the beginning. This is because in actual use, these four timings are only used immediately after other timings **except the Stop Signal timing**, and those timings all restore SCL to a low level at the end, so it does not matter.
<br>



## 3.2 Receive Acknowledge Timeout Optimization
This is the most passive timing for the master. Continuous communication requires waiting for the slave to complete its internal work and provide a response; otherwise, the slave cannot continue. If the master chooses to wait for the slave’s response, and if the slave is faulty or not actually connected, the master process may get stuck in a long wait.
<br>You can set a maximum waiting time. If the slave responds before the timeout, communication continues; otherwise, it stops. The timeout detection is implemented through delay plus polling. The improved code reference is as follows:
```C++
u8 IIC_ReceiveACK(void) {
    u8 ACK, TimeOut=200;    // <-- The initial value of TimeOut can be changed as needed

    SDA = 1;
    SCL = 1;
    while(TimeOut--) {
        ACK = SDA;
        if(!ACK) {break;}   // <-- Acknowledge (ACK = 0) is used here as the sign that the slave is ready

        IIC_Delay10us();    // <-- The delay function can be modified here, but too long a delay will seriously affect the communication speed. In this example, assuming a 10us delay, the code will check SDA every 10us. If an acknowledge is received, it breaks out of the polling loop early, and the function eventually returns acknowledge (return 0); otherwise, the polling loop ends after 200 iterations, with a total timeout of 200 * 10us = 2ms, and the function returns not-acknowledge (return 1).

    }
    SCL = 0;
    return ACK;
}
```
<br>



## 3.3 Communication Speed Adaptation Optimization
Due to the RC characteristics introduced by device interfaces and wiring, the level on the bus requires a certain amount of time to stabilize after being toggled by the master or slave. The logic processing unit of the slave also has an upper limit on signal processing speed. Both are the main factors limiting IIC communication speed.
<br>To ensure both sufficient speed and reliability of communication, in addition to keeping wiring as short as possible, protecting the bus from electromagnetic interference, and selecting appropriate pull-up resistors based on the bus load, it is also necessary to actively limit the pin operation speed from the microcontroller program.
<br>Taking the standard mode of 100kHz supported by most devices as an example, you can add a 10us delay after each pin level change in the code. The pin functions can be encapsulated as follows:
```C++
void IIC_Delay10us(void) {
    // 10us delay code
}

void IIC_EditSCL(u8 Dat) {
    if(Dat) {SCL = 1;}
    else {SCL = 0;}
    IIC_Delay10us();
}

void IIC_EditSDA(u8 Dat) {
    if(Dat) {SDA = 1;}
    else {SDA = 0;}
    IIC_Delay10us();
}
```
<br>



## 3.4 Final Code Reference
```C++
// Delay function for controlling the level transition speed
void IIC_Delay10us(void) {
    // 10us delay code
}



// Level control wrapper function
void IIC_EditSCL(u8 Dat) {
    if(Dat) {SCL = 1;}
    else {SCL = 0;}
    IIC_Delay10us();
}
void IIC_EditSDA(u8 Dat) {
    if(Dat) {SDA = 1;}
    else {SDA = 0;}
    IIC_Delay10us();
}



// Start Signal
void IIC_Start(void) {
    IIC_EditSDA(1);
    IIC_EditSCL(1);
    IIC_EditSDA(0);
    IIC_EditSCL(0);
}

// Stop Signal
void IIC_Stop(void) {
    IIC_EditSCL(0);
    IIC_EditSDA(0);
    IIC_EditSCL(1);
    IIC_EditSDA(1);
}

// Send Acknowledge
void IIC_SendACK(u8 ACK) {
    IIC_EditSDA(ACK);
    IIC_EditSCL(1);
    IIC_EditSCL(0);
}

// Receive Acknowledge
u8 IIC_ReceiveACK(void) {
    u8 ACK, TimeOut=200;

    IIC_EditSDA(1);
    IIC_EditSCL(1);
    while(TimeOut--) {
        ACK = SDA;
        if(!ACK) {break;}
        IIC_Delay10us();
    }
    IIC_EditSCL(0);
    return ACK;
}

// Send Byte
void IIC_SendByte(u8 Byte) {
    u8 i;

    for(i = 0 ; i < 8 ; i++) {
        IIC_EditSDA(Byte & 0x80);
        IIC_EditSCL(1);
        IIC_EditSCL(0);
        Byte = Byte << 1;
    }
}

// Receive Byte
u8 IIC_ReceiveByte(void) {
    u8 i, Byte=0x00;

    IIC_EditSDA(1);
    for(i = 0 ; i < 8 ; i++) {
        Byte = Byte << 1;
        IIC_EditSCL(1);
        if(SDA) {Byte |= 0x01;}
        IIC_EditSCL(0);
    }
    return Byte;
}
```
