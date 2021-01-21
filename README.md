# moisture sensor
This guide is a continuation of the `Dragino Lora Shield Guide` and will make use of the application and device set up during the guide.
## What you will need
To follow this guide, you will need the following:
- An [Arduino Uno](https://www.jaycar.com.au/duinotech-uno-r3-development-board/p/XC4410) or [Arduino Mega](https://www.jaycar.com.au/duinotech-mega-2560-r3-board-for-arduino/p/XC4420)
- A [Dragino Lora Shield](https://www.jaycar.com.au/arduino-compatible-long-range-lora-shield/p/XC4392) for the Arduino
- A [I2C Soil moisture sensor](https://www.tindie.com/products/miceuz/i2c-soil-moisture-sensor/)
- 4 wires
- A computer to connect to the Arduino and write the code
- A usb A to usb B cable to connect the Arduino to the your computer

You will also need to be in range of a Gateway connected to The Things Network which you can find out about [here](https://www.thethingsnetwork.org/community).

## Step 1 - Install Library
- Install the library by going to `Tools > Manage Libraries`
- Search for `I2CSoilMoistureSensor` using the searchbox
- Install the `I2CSoilMoistureSensor` library

## step 2 - setup the board
- First attach the Lora Shield to the Arduino by slotting the shields bottom pins into the Arduino ports, ensuring that the pins labels on the shield match with the port labels on the Arduino (e.g., the GND pin on the shield enters the GND port on the Arduino).

- Once the Lora shield is attached to the Arduino take the antenna that came with the Lora shield and attach it to the metal input on the shield and fasten the hand screw to fix it in place.

- Using a wire connect one of the GND ports on the Dragino Lora Shield to the GND port on the I2C Soil moisture sensor.
- Using a wire connect one of the SCL ports on the Dragino Lora Shield to the SCK/SCL port on the I2C Soil moisture sensor.
- Using a wire connect one of the SDA ports on the Dragino Lora Shield to the SDA/MOSI port on the I2C Soil moisture sensor.
- Using a wire connect one of the 5V ports on the Dragino Lora Shield to the VCC port on the I2C Soil moisture sensor.

## step 3 - write the code
Now that the sensor is connected to the Arduino, the code needs to be modified from the `Dragino Lora Shield Guide` to send the sensor data instead of the static letters in the myData variable.

- Under the line `#include <SPI.h>` at the top of the code add the following lines
```
#include <I2CSoilMoistureSensor.h>
#include <Wire.h>
I2CSoilMoistureSensor sensor;
```
- In the `void setup()` function add `Wire.begin();` above the `Serial.begin(9600);` line
- Above the `//LMIC init` line add the following code
```
sensor.begin(); // reset sensor
delay(1000); // give some time to boot up
Serial.print("I2C Soil Moisture Sensor Address: ");
Serial.println(sensor.getAddress(),HEX);
Serial.print("Sensor Firmware version: ");
Serial.println(sensor.getVersion(),HEX);
Serial.println();
```
This will reset the soil moisture sensor and display the I2C address and firmware version in the serial monitor

- Next, in the `void do_send(osjob_t* j)` function add the following code below the `} else {` line
```
        while (sensor.isBusy()) delay(50);
        int soilMoistureCapacitance = sensor.getCapacitance();
        int temperature = sensor.getTemperature();
        int light = sensor.getLight(true);
        sensor.sleep(); //sleep the sensor to save power
        
        Serial.print("Soil Moisture Capacitance: ");
        Serial.print(soilMoistureCapacitance); //read capacitance register
        Serial.print(", Temperature: ");
        Serial.print(temperature /(float)10); //temperature register
        Serial.print(", Light: ");
        Serial.println(light); //request light measurement, wait and read light register
  
        //Break the soilMoistureCapacitance, temperature and light into Bytes in individual buffer arrays
        byte payloadA[2];
        payloadA[0] = highByte(soilMoistureCapacitance);
        payloadA[1] = lowByte(soilMoistureCapacitance);
        byte payloadB[2]; 
        payloadB[0] = highByte(temperature);
        payloadB[1] = lowByte(temperature);
        byte payloadC[2]; 
        payloadC[0] = highByte(light);
        payloadC[1] = lowByte(light);

        //Get the size of each buffer array (in this case it will always be 2, but this could be used if they were variable)
        int sizeofPayloadA = sizeof(payloadA);
        int sizeofPayloadB = sizeof(payloadB);
        int sizeofPayloadC = sizeof(payloadC);

        //Make a buffer array big enough for all the values
        byte payload[sizeofPayloadA + sizeofPayloadB + sizeofPayloadC];
        
        //Add each of the individual buffer arrays the single large buffer array
        memcpy(payload, payloadA, sizeofPayloadA);
        memcpy(payload + sizeofPayloadA, payloadB, sizeofPayloadB);
        memcpy(payload + sizeofPayloadA + sizeofPayloadB, payloadC, sizeofPayloadC);
```
### What does this code do?
This code will first wait until the sensor is not busy. Then it will record the soil moisture capacitance, temperature, and light readings from the sensor.
Next it will display the values in the serial monitor so they can be checked easily.
Finally, the code goes through a process to convert the values from numbers into a payload made of a buffer array that can be sent to The Things Network. Find out more about this process [here](https://www.thethingsnetwork.org/docs/devices/bytes.html).

- Finally, replace the line `LMIC_setTxData2(1, mydata, sizeof(mydata)-1, 0);` with the line `LMIC_setTxData2(1, payload, sizeof(payload), 0);` in order to send the newly created payload instead of the static "Hello, world!" text to The Things Network.

## Step 4 - Decoding the message
Now that we have encoded the message and sent it to The Things Network we need to tell The Things Network what to do with it.

- In your application on The Things Network, go to the tab named `Payload Formats`

- In here we can write code to decrypt the data we get from our device.
- Enter the following into the decoder
```
function Decoder(bytes, port) {
  // Decode an uplink message from a buffer
  var decoded = {};
  //Separate the individual numbers into individual buffers
  var Val1 = bytes.slice(0, 2);
  var Val2 = bytes.slice(2, 4);
  var Val3 = bytes.slice(4, 6);
  
  //Convert the buffers into numbers and divide by 100 to return them to 2 decimal places
  //Then save them to the decoded array
  decoded.myValA = ((Val1[0] << 8) + Val1[1]);
  decoded.myValB = ((Val2[0] << 8) + Val2[1]) / 10;
  decoded.myValC = ((Val3[0] << 8) + Val3[1]);

  //return the values
  return {
    field1: decoded.myValA,
    field2: decoded.myValB,
    field3: decoded.myValC,
  };
}
```
The Code first separates the long buffer array we created in step 3 into the buffers that make up the soil moisture capacitance, temperature, and light readings.
It then decodes each of the buffers into numbers, divides the temperature by 10 to turn it into a decimal number and saves them into the decoded array.
Finally the numbers are returned as field 1, field 2 and field 3.

- Click the `Save payload functions` button

## Step 5 - Testing everything
If everything up to this point has gone well, all that’s left to do is start the program.
- Connect the Arduino to your computer using the usb cable.
- In the Arduino IDE ensure that the correct port is selected by going to `Tools > Port:` and check that the Arduino's is selected
- Ensure that the correct board is selected by going to `Tools > Board: > Arduino AVR Boards` and selecting the type of board you have
- Finally click the arrow button in the top left to upload your code to the Arduino

After it has finished uploading you can check the monitor at `Tools > Serial Monitor` to see if it is working.
You can also now go to the `Data` tab on your The Things Network application to see the data being sent.