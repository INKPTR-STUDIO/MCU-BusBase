# Table of Contents
1. Physical Bus
2. Communication Timing

    - 2.1 Initialization Function
    - 2.2 Start Signal
    - 2.3 Stop Signal
    - 2.4 Mode 0 Byte Transfer(select one of the four modes)
    - 2.5 Mode 0 Byte Transfer(select one of the four modes)
    - 2.6 Mode 0 Byte Transfer(select one of the four modes)
    - 2.7 Mode 0 Byte Transfer(select one of the four modes)

3. Supplementary Notes and Final Reference Code

    - 3.1 Modes 0 & 1 and Modes 2 & 3
    - 3.2 Communication Speed Adaptation Optimization
    - 3.3 Final Reference Code

<br><br><br>






# 1. Physical Bus
The SPI physical bus consists of four lines:
<br>(1) Chip Select (CS) – Controlled by the master to select a target slave. Also written as SS, CS#, etc. (CS# emphasizes active low).
<br>(2) Serial Clock (SCK) – Provides the clock signal from the master. Also written as SCL.
<br>(3) Slave Data Input (SI) – Used by the master to send commands or data to a slave. Also written as DI, MOSI. (MOSI emphasizes the direction: Master Out, Slave In.)
<br>(4) Slave Data Output (SO) – Used by a slave to send responses or data back to the master. Also written as DO, MISO. (MISO emphasizes the direction: Master In, Slave Out.)
<br>For the master, CS, SCK, and SI are configured as push-pull outputs, while SO is configured as a floating input. It is recommended to add a pull-up resistor on CS to stabilize the CS level during power-up and prevent noise from causing false triggering of the slave. SCK, SI, and SO can be connected directly. (In typical 3.3 V–5 V supply scenarios, a 10 kΩ pull-up resistor is recommended. Note: if a slave’s SO pin is open-drain, a pull-up resistor is also required on SO. For details, refer to the datasheet of the device you are using.)
<br>Each slave requires an independent CS line to control its selection; SCK, SI, and SO are connected in parallel to multiple slaves. Normally, the master pulls low the CS of the intended slave to bring it into communication, while slaves not selected will not respond.
<br>All data lines are unidirectional: CS, SCK, and SI go from master to slave; SO goes from slave to master. In scenarios where the master only outputs data, the SO line may be omitted.
<br>![SPI_Bus](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Bus.png)
<br><br><br>






# 2. Communication Timing
As described above, the slave only needs to decide whether to enter communication based on the CS level. However, depending on how the clock and data lines work together, SPI has four different communication modes. Refer to the datasheet of your device to determine which mode to use.
- Mode 0: SCK idles low, and the master changes the SI level; the slave receives data on the SCK rising edge and sends data on the falling edge.
- Mode 1: SCK idles low, and the master changes the SI level; the slave sends data on the SCK rising edge and receives data on the falling edge.
- Mode 2: SCK idles high, and the master changes the SI level; the slave receives data on the SCK falling edge and sends data on the rising edge.
- Mode 3: SCK idles high, and the master changes the SI level; the slave sends data on the SCK falling edge and receives data on the rising edge.

After being left at default settings or properly configured, a device will adhere to only one of these modes and will not switch; simply choose one to use. Moreover, different initialization functions are required depending on the SCK idle level requirements of each mode. Therefore, driving an SPI peripheral requires a total of one initialization function and three timing sequences: start condition, stop condition, and byte exchange. By combining the required timing sequences according to the device’s communication requirements, SPI communication can be realized. The timing details will be explained in the following sections.
<br>Here it is assumed that the prefix "T_" denotes host bytes sent by the host, and "R_" denotes slave bytes received by the host. Same below.
<br>![SPI_Timing](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Timing.png)
<br>



## 2.1 Initialization Function
*Prevents false triggering of the slave due to undefined logic levels that could cause the first communication to fail, while also initializing the bus levels to a known state.*
<br>
<br>Logic requirement: Drive CS **high**. For Mode 0 or 1, drive SCK **low**; for Mode 2 or 3, drive SCK **high**.
<br>
Reference code for Mode 0 or 1:
```C++
void SPI_Init(void) {
    CS = 1;
    SCK = 0;
}
```
Reference code for Mode 2 or 3:
```C++
void SPI_Init(void) {
    CS = 1;
    SCK = 1;
}
```
<br>



## 2.2 Start Signal
*Tell the device that communication is starting.*
<br>
<br>Logic requirement: Set CS **low**.
<br>
Code reference:
```C++
void SPI_Start(void) {
    CS = 0;
}
```
<br>



## 2.3 Stop Signal
*Tell the device that communication is ending.*
<br>
<br>Logic requirement: Set CS **high**.
<br>
Code reference:
```C++
void SPI_Stop(void) {
    CS = 1;
}
```
<br>



## 2.4 Mode 0 Byte Transfer(select one of the four modes)
*Send a byte to the slave while simultaneously receiving a byte from the slave. The same applies below.*
<br>
![SPI_Mode0](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Mode0.png)
<br>Logic requirement: Before SCK generates a **rising edge**, the bit of the byte to be sent must be ready on SI. After SCK generates a rising edge, the level on SO is sampled, and then a **falling edge** is generated on SCK. Bytes are sent and received MSB-first, repeating 8 times to complete one byte transfer. The same applies below (i.e., D7, D6, D5, …, D0).
<br>Example: Assume the master sends the byte 0xCA (binary 1100 1010). Each time SCK generates a falling edge, the SI level will be, in sequence: 1, 1, 0, 0, 1, 0, 1, 0. Assume the slave returns the byte 0xA2 (binary 1010 0010). After each SCK rising edge, the SO level will be, in sequence: 1, 0, 1, 0, 0, 0, 1, 0.

|Step|Level Operation|Purpose|
|:-:|:-:|:-|
|1|SI = T_D7|Prepare the master byte D7 bit level on SI|
|2|SCK = 1|Generate a rising edge on SCK|
|3|R_D7 = SO|Sample the SO level as the slave byte D7 bit|
|4|SCK = 0|Generate a falling edge on SCK|
|5|Repeat steps 1–4, with SI = T_D6, R_D6 = SO|Send master byte D6 while receiving slave byte D6|
|6|Repeat steps 1–4, with SI = T_D5, R_D5 = SO|Send master byte D5 while receiving slave byte D5|
|...|...|...|
|11|Repeat steps 1–4, with SI = T_D0, R_D0 = SO|Send master byte D0 while receiving slave byte D0|

Code reference:
```C++
u8 SPI_TransferByte(u8 SendByte) {    // <-- u8 means unsigned char, same below
    u8 i, ReceiveByte=0x00;

    for(i = 0 ; i < 8 ; i++) {
        ReceiveByte = ReceiveByte << 1;
        if(SendByte & 0x80) {SI = 1;}   // <-- This uses the truth value of (SendByte & 0x80), i.e., it checks whether the result is 0 or non-zero. Directly writing SI = SendByte & 0x80 is not suitable and may cause abnormal pin register operation, especially when (SendByte & 0x80) > 1. Same below.
        else {SI = 0;}
        SCK = 1;
        if(SO) {ReceiveByte |= 0x01;}
        SCK = 0;
        SendByte = SendByte << 1;
    }
    return ReceiveByte;
}
```
<br>



## 2.5 Mode 1 Byte Transfer(select one of the four modes)
![SPI_Mode0](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Mode0.png)
<br>Logic requirement: After generating a **rising edge** on SCK, prepare the next bit on SI before generating a **falling edge**; after the falling edge, sample SO.

|Step|Level Operation|Purpose|
|:-:|:-:|:-|
|1|SCK = 1|Generate a rising edge on SCK|
|2|SI = T_D7|Prepare the master byte D7 bit level on SI|
|3|SCK = 0|Generate a falling edge on SCK|
|4|R_D7 = SO|Sample the SO level as the slave byte D7 bit|
|5|Repeat steps 1–4, with SI = T_D6, R_D6 = SO|Send master byte D6 while receiving slave byte D6|
|6|Repeat steps 1–4, with SI = T_D5, R_D5 = SO|Send master byte D5 while receiving slave byte D5|
|...|...|...|
|11|Repeat steps 1–4, with SI = T_D0, R_D0 = SO|Send master byte D0 while receiving slave byte D0|

Code reference:
```C++
u8 SPI_TransferByte(u8 SendByte) {
    u8 i, ReceiveByte=0x00;

    for(i = 0 ; i < 8 ; i++) {
        ReceiveByte = ReceiveByte << 1;
        SCK = 1;
        if(SendByte & 0x80) {SI = 1;}
        else {SI = 0;}
        SCK = 0;
        if(SO) {ReceiveByte |= 0x01;}
        SendByte = SendByte << 1;
    }
    return ReceiveByte;
}
```
<br>



## 2.6 Mode 2 Byte Transfer(select one of the four modes)
![SPI_Mode0](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Mode0.png)
<br>Logic requirement: Before SCK generates a **falling edge**, the bit of the byte to be sent must be ready on SI. After SCK generates a falling edge, the level on SO is sampled, and then a **rising edge** is generated on SCK.

|Step|Level Operation|Purpose|
|:-:|:-:|:-|
|1|SI = T_D7|Prepare the master byte D7 bit level on SI|
|2|SCK = 0|Generate a falling edge on SCK|
|3|R_D7 = SO|Sample the SO level as the slave byte D7 bit|
|4|SCK = 1|Generate a rising edge on SCK|
|5|Repeat steps 1–4, with SI = T_D6, R_D6 = SO|Send master byte D6 while receiving slave byte D6|
|6|Repeat steps 1–4, with SI = T_D5, R_D5 = SO|Send master byte D5 while receiving slave byte D5|
|...|...|...|
|11|Repeat steps 1–4, with SI = T_D0, R_D0 = SO|Send master byte D0 while receiving slave byte D0|

Code reference:
```C++
u8 SPI_TransferByte(u8 SendByte) {
    u8 i, ReceiveByte=0x00;

    for(i = 0 ; i < 8 ; i++) {
        ReceiveByte = ReceiveByte << 1;
        if(SendByte & 0x80) {SI = 1;}
        else {SI = 0;}
        SCK = 0;
        if(SO) {ReceiveByte |= 0x01;}
        SCK = 1;
        SendByte = SendByte << 1;
    }
    return ReceiveByte;
}
```
<br>



## 2.7 Mode 3 Byte Transfer(select one of the four modes)
![SPI_Mode0](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Mode0.png)
<br>Logic requirement: After generating a **falling edge** on SCK, prepare the next bit on SI before generating a **rising edge**; after the rising edge, sample SO.

|Step|Level Operation|Purpose|
|:-:|:-:|:-|
|1|SCK = 0|Generate a falling edge on SCK|
|2|SI = T_D7|Prepare the master byte D7 bit level on SI|
|3|SCK = 1|Generate a rising edge on SCK|
|4|R_D7 = SO|Sample the SO level as the slave byte D7 bit|
|5|Repeat steps 1–4, with SI = T_D6, R_D6 = SO|Send master byte D6 while receiving slave byte D6|
|6|Repeat steps 1–4, with SI = T_D5, R_D5 = SO|Send master byte D5 while receiving slave byte D5|
|...|...|...|
|11|Repeat steps 1–4, with SI = T_D0, R_D0 = SO|Send master byte D0 while receiving slave byte D0|

Code reference:
```C++
u8 SPI_TransferByte(u8 SendByte) {
    u8 i, ReceiveByte=0x00;

    for(i = 0 ; i < 8 ; i++) {
        ReceiveByte = ReceiveByte << 1;
        SCK = 0;
        if(SendByte & 0x80) {SI = 1;}
        else {SI = 0;}
        SCK = 1;
        if(SO) {ReceiveByte |= 0x01;}
        SendByte = SendByte << 1;
    }
    return ReceiveByte;
}
```
<br><br><br>






# 3. Supplementary Notes and Final Reference Code
## 3.1 Modes 0 & 1 and Modes 2 & 3
Compared to the logic requirement descriptions under the heading "2.x Mode x Byte Transfer(select one of the four modes)", the requirements described in the "2. Communication Timing" section regarding how the clock and data lines work together may appear more relaxed. This can lead to a misunderstanding: viewing the SCK rising and falling edges together as a single pulse, preparing the data to be sent before the pulse, and sampling the received data after the pulse. This might work when the master operates at a very low speed, but errors can easily occur at slightly higher communication speeds.
<br>
<br>For both the master and the slave, the SCK edge serves only as a trigger to sample the data line level; the actual sampling does not happen exactly at that edge. The level that is actually sampled comes from a very short time after the SCK edge. In other words, the actual sampling moment can be considered to occur during the level-stable period following an SCK edge, not at the edge itself. ("A very short time after the SCK edge": This time is relatively fixed and is determined by the device structure.)
<br>In different modes, the valid data output windows for the master and the slave overlap during one of the SCK level periods:
- Mode 0: SI and SO are allowed to change during SCK low. (In this mode, SCK idles low, so changes are allowed during the idle period.)
- Mode 1: SI and SO are allowed to change during SCK high. (In this mode, SCK idles low, so changes are not allowed during the idle period.)
- Mode 2: SI and SO are allowed to change during SCK high. (In this mode, SCK idles high, so changes are allowed during the idle period.)
- Mode 3: SI and SO are allowed to change during SCK low. (In this mode, SCK idles high, so changes are not allowed during the idle period.)

When the master is updating SI, the slave is also updating SO. Placing the SO sampling code and the SI update code together may result in sampling an incorrect level, either because the SO level has not yet stabilized after the slave has just updated it, or because the slave has not even updated SO yet. At very low master speeds, the sampling moment may coincidentally fall within the SO level-stable period, which is why it might succeed. However, one of the key advantages of the SPI protocol is communication speed, and this trade-off is not worthwhile.
<br>
<br>Therefore, the description in the "2. Communication Timing" section should only serve as an aid for mode matching. (Some datasheets may not explicitly state which SPI mode a device uses, but instead describe on which edge it operates in a certain way.)
<br>



## 3.2 Communication Speed Adaptation Optimization
As mentioned in “Modes 0 & 1 and Modes 2 & 3”, due to the RC characteristics introduced by device interfaces and wiring, after a level is toggled by the master or slave, it takes some time for the level on the bus to stabilize. The slave’s logic processing unit also has an upper limit on signal processing speed. These two factors are the main constraints on SPI communication speed. (This applies unless the master’s own level switching speed is already slow enough. If the master speed is sufficiently low, the optimization described in this section can be omitted.)
<br>To achieve both adequate speed and reliability, in addition to keeping wiring as short as possible, protecting the bus from electromagnetic interference, and selecting appropriate pull-up resistors based on the bus load, it is also necessary to actively limit the pin operation speed in the MCU program.
<br>Taking 1 MHz, which is supported by most devices, as an example, a 250 ns delay can be inserted before and after each pin level change operation. The reference code can wrap the pin functions as follows:
```C++
void SPI_Delay250ns(void) {
    // 250ns delay code
}

void SPI_EditSCK(u8 Dat) {
    SPI_Delay250ns();
    if(Dat) {SCK = 1;}
    else {SCK = 0;}
    SPI_Delay250ns();
}
```
If the MCU pin toggling speed is already lower than 1 MHz, this delay can be omitted.
<br>
<br>Furthermore, during the start and stop conditions, the slave needs time to fully enter the communication state after CS is pulled low, and also needs time to fully exit the communication state after CS is pulled high. Therefore, a delay must be added **after pulling CS low** and **before pulling CS high**. The specific delay requirements vary by device; refer to the datasheet of the device you are using for details.
<br>Taking 1 µs, which is supported by most devices, as an example, the following pin functions can be wrapped:
```C++
void SPI_Delay1us(void) {
    // 1us delay code
}

void SPI_Start(void) {
    CS = 0;
    SPI_Delay1us();
}

void SPI_Stop(void) {
    SPI_Delay1us();
    CS = 1;
}
```
<br>



## 3.3 Final Reference Code
```C++
#ifndef SPI_H
#define SPI_H

void SPI_Init(void);                // Initialization Function
void SPI_Start(void);               // Start Signal
void SPI_Stop(void);                // Stop Signal
u8 SPI_TransferByte(u8 SendByte);   // Byte Transfer

#endif
```
```C++
// Delay Function
static void SPI_Delay1us(void) {
    // 1us delay code
}
static void SPI_Delay250ns(void) {
    // 250ns delay code
}



// SCK wrapper function
static void SPI_EditSCK(u8 Dat) {
    SPI_Delay250ns();
    if(Dat) {SCK = 1;}
    else {SCK = 0;}
    SPI_Delay250ns();
}



// Initialization Function
void SPI_Init(void) {
    CS = 1;
    SCK = 0;    // <-- SCK = 0 in Mode 0/1; SCK = 1 in Mode 2/3.
}

// Start Signal
void SPI_Start(void) {
    CS = 0;
    SPI_Delay1us();
}

// Stop Signal
void SPI_Stop(void) {
    SPI_Delay1us();
    CS = 1;
}

// Mode 0 Byte Transfer
u8 SPI_TransferByte(u8 SendByte) {
    u8 i, ReceiveByte=0x00;

    for(i = 0 ; i < 8 ; i++) {
        ReceiveByte = ReceiveByte << 1;
        if(SendByte & 0x80) {SI = 1;}
        else {SI = 0;}
        SPI_EditSCK(1);
        if(SO) {ReceiveByte |= 0x01;}
        SPI_EditSCK(0);
        SendByte = SendByte << 1;
    }
    return ReceiveByte;
}

// Mode 1 Byte Transfer
//u8 SPI_TransferByte(u8 SendByte) {
//    u8 i, ReceiveByte=0x00;
//
//    for(i = 0 ; i < 8 ; i++) {
//        ReceiveByte = ReceiveByte << 1;
//        SPI_EditSCK(1);
//        if(SendByte & 0x80) {SI = 1;}
//        else {SI = 0;}
//        SPI_EditSCK(0);
//        if(SO) {ReceiveByte |= 0x01;}
//        SendByte = SendByte << 1;
//    }
//    return ReceiveByte;
//}

// Mode 2 Byte Transfer
//u8 SPI_TransferByte(u8 SendByte) {
//    u8 i, ReceiveByte=0x00;
//
//    for(i = 0 ; i < 8 ; i++) {
//        ReceiveByte = ReceiveByte << 1;
//        if(SendByte & 0x80) {SI = 1;}
//        else {SI = 0;}
//        SPI_EditSCK(0);
//        if(SO) {ReceiveByte |= 0x01;}
//        SPI_EditSCK(1);
//        SendByte = SendByte << 1;
//    }
//    return ReceiveByte;
//}

// Mode 3 Byte Transfer
//u8 SPI_TransferByte(u8 SendByte) {
//    u8 i, ReceiveByte=0x00;
//
//    for(i = 0 ; i < 8 ; i++) {
//        ReceiveByte = ReceiveByte << 1;
//        SPI_EditSCK(0);
//        if(SendByte & 0x80) {SI = 1;}
//        else {SI = 0;}
//        SPI_EditSCK(1);
//        if(SO) {ReceiveByte |= 0x01;}
//        SendByte = SendByte << 1;
//    }
//    return ReceiveByte;
//}
```