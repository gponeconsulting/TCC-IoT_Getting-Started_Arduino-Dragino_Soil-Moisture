# Moisture Sensor
This guide is a continuation of the `Getting Started with the Arduino and Dragino Guide` and will make use of the application (on The Things Network) and device (physical Arduino + Dragino) set up during that guide. If you haven't followed the previous guide you will need to do that first, and  have:
- The Arduino and Dragino connected correctly, with an antenna
- The Arduino connected your computer, and communicating correctly
- The Arduino IDE installed
- The `MCCI LoRaWAN LMIC library` installed and configured correctly, and
- The Arduino/Dragino registered and communicating with The Things Network

## What you will need
To follow this guide, you will The setup from the previous guide, with the addition of:
- An [I2C Soil moisture sensor](https://www.tindie.com/products/miceuz/i2c-soil-moisture-sensor/).
- 4 jump wires.

You will still need to be in range of a Gateway connected to The Things Network which you can find out about [here](https://www.thethingsnetwork.org/community).

## Step 1 - Install Library
Install a library to help use the soil moisture sensor.

In the Arduino IDE:
- Go to `Tools -> Manage Libraries`
- Search for `I2CSoilMoistureSensor` using the search box
- Install the `I2CSoilMoistureSensor` library

![Install Soil Moisture Library](readme-images/library.png)

## Step 2 - Setup the Board
Building on the setup from the setup in Getting Started with the Arduino and Dragino guide:

1. Use a wire to connect one of the GND ports on the Dragino Lora Shield to the GND port on the I2C Soil moisture sensor.
1. Use a wire to connect the SCL port on the Dragino Lora Shield to the SCK/SCL port on the I2C Soil moisture sensor.
1. Use a wire to connect the SDA port on the Dragino Lora Shield to the SDA/MOSI port on the I2C Soil moisture sensor.
1. Use a wire to connect the 5V port on the Dragino Lora Shield to the VCC port on the I2C Soil moisture sensor.

![Arduino Physical Setup](readme-images/arduino-physical-setup.jpg)

## Step 3 - The Arduino Code
Now that the sensor is connected to the Arduino, the code needs to be modified from the `Dragino Lora Shield Guide` to send the sensor data instead of the static letters in the myData variable.

> Make sure you get the code from the `Dragino Lora Shield Guide` which importantly has the APPEUI, DEVEUI, APPKEY and Pin Mapping configuration correct.

Under the line `#include <SPI.h>` at the top of the code add the following lines

```C++
#include <I2CSoilMoistureSensor.h>
#include <Wire.h>
I2CSoilMoistureSensor sensor;
```

In the `void setup()` function add `Wire.begin();` above the `Serial.begin(9600);` line:

```C++
void setup() {
    Wire.begin();
    Serial.begin(9600);
    Serial.println(F("Starting"));
    ...
}
```

Above the `//LMIC init` line add the following code:

```C++
sensor.begin(); // reset sensor
delay(1000); // give some time to boot up
Serial.print("I2C Soil Moisture Sensor Address: ");
Serial.println(sensor.getAddress(),HEX);
Serial.print("Sensor Firmware version: ");
Serial.println(sensor.getVersion(),HEX);
Serial.println();
```

>This will reset the soil moisture sensor and display the I2C address and firmware version in the serial monitor

Next, in the `void do_send(osjob_t* j)` function replace:

```C++
// Prepare upstream data transmission at the next possible time.
LMIC_setTxData2(1, mydata, sizeof(mydata)-1, 0);
```

with the following code:
```C++
//Wait until the sensor is not busy
while (sensor.isBusy()) delay(50);

//Record the soil moisture capacitance, temperature, and light readings
int soilMoistureCapacitance = sensor.getCapacitance();
int temperature = sensor.getTemperature();
int light = sensor.getLight(true);
sensor.sleep(); //sleep the sensor to save power

//Display the values in the serial monitor for reference
Serial.print("Soil Moisture Capacitance: ");
Serial.print(soilMoistureCapacitance); //read capacitance register
Serial.print(", Temperature: ");
Serial.print(temperature /(float)10); //temperature register
Serial.print(", Light: ");
Serial.println(light); //request light measurement, wait and read light register

//Convert the values from numbers into a payload
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

//Get the size of each buffer array (in this case they will always be 2, but this could be used if they were variable)
int sizeofPayloadA = sizeof(payloadA);
int sizeofPayloadB = sizeof(payloadB);
int sizeofPayloadC = sizeof(payloadC);

//Make a buffer array big enough for all the values
byte payload[sizeofPayloadA + sizeofPayloadB + sizeofPayloadC];

//Add each of the individual buffer arrays the single large buffer array
memcpy(payload, payloadA, sizeofPayloadA);
memcpy(payload + sizeofPayloadA, payloadB, sizeofPayloadB);
memcpy(payload + sizeofPayloadA + sizeofPayloadB, payloadC, sizeofPayloadC);

// Prepare upstream data transmission at the next possible time.  
LMIC_setTxData2(1, payload, sizeof(payload), 0);
```

### What does this code do?
This code will
1. First wait until the sensor is not busy.
1. Then it will record the soil moisture capacitance, temperature, and light readings from the sensor.
1. Next it will display the values in the serial monitor so they can be checked easily.
1. The code converts the values from numbers into a payload made of a buffer array that can be sent to The Things Network. To find out more about this process [here](https://www.thethingsnetwork.org/docs/devices/bytes.html).
1. Finally the newly created payload is queued for transmission (instead of the static "Hello, world!" text to The Things Network.)

### Testing
1. Connect the Arduino to your computer using the USB cable.
1. In the Arduino IDE ensure that the correct port is selected by going to `Tools -> Port:` and check that the Arduino's is selected
1. Ensure that the correct board is selected by going to `Tools -> Boards: <..> -> Arduino AVR Boards` and selecting the type of board you have
1. Finally click the arrow button in the top left to upload your code to the Arduino.
1. You should see a 'successful upload' message in the bottom of the Arduino IDE

After it has finished uploading you can check the monitor at `Tools -> Serial Monitor` to see if it is working. You should see it connect to The Things Network, make measurements and send those measurements.

You can also now go to the `Data` tab on your The Things Network application to see the data being sent.

*Remember: Don't be worried if it fails to connect a few times*

![Connect & Measure Successful](readme-images/connect-and-send.png)

## Step 4 - Decoding the message
Now that we have encoded the message and sent it to The Things Network we need to tell The Things Network what to do with it.

- In your application on The Things Network, go to the tab named `Payload Formats`
- In here we can write code to decrypt the data we get from our device.
- Enter the following into the decoder

```C++
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

![Decode](readme-images/decoder.png)

The Code first separates the long buffer array we created in Step 3 into the buffers that make up the soil moisture capacitance, temperature, and light readings.

It then decodes each of the buffers into numbers, divides the temperature by 10 to turn it into a decimal number and saves them into the decoded array.

Finally the numbers are returned as field 1, field 2 and field 3.

- Click the `Save payload functions` button

### The Decoded Message

You can also now go to the `Data` tab on your The Things Network application to see the data being sent, just like before, but now the "decoded" values are shown as well.

![Decoded Payload](readme-images/decoded-payload.jpg)
