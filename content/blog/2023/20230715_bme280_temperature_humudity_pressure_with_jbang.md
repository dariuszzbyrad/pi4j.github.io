---
title: BME280 (temp, humidity, pressure) via Pi4J, I2C, and JBang
date: 2023-07-15

---

2023-07-07, by Frank Delporte and Tom Aarts

{{% notice note %}}
GITHUB PROJECT: [https://github.com/Pi4J/pi4j-jbang](https://github.com/Pi4J/pi4j-jbang)
{{% /notice %}}

## Intro

This is an example project to demonstrate how to read the temperature, humidity and pressure from a BME280 sensor, installed on an Adafruit board that can be controlled via I2C and SPI.

## Wiring

As this board can be controlled with both I2C and SPI, there are more available connections than we need for I2C. Make the following connections between the BME280 and a Raspberry Pi:

* Vin to 3.3V
* GND to GND
* SCK to I2C clock SCL (pin 5)
* SDI to I2C data SDA (pin 3)

In the wiring diagram, another brand of board ([Sparkfun](https://www.sparkfun.com/products/13676)) is used, which has the I2C and SPI connections on different sides, but to align it with the pictures, the same connections are used as are available on the Adafruit board.

{{< gallery >}}
{{< figure link="/assets/blogs/2023/bme280_wiring_i2c.jpg" caption="BME280 I2C wiring" caption-position="center" caption-effect="fade" >}}
{{< figure link="/assets/blogs/2023/bme280_wiring_i2c_rpi.jpg" caption="BME280 I2C wiring" caption-position="center" caption-effect="fade" >}}
{{< figure link="/assets/blogs/2023/bme280_wiring_i2c_breadboard.png" caption="BME280 I2C wiring" caption-position="center" caption-effect="fade" >}}
{{< /gallery >}}
{{< load-photoswipe >}}

### Check the Wiring

After the wiring has been completed, you can check if the device is connected correctly by checking if the I2C device is detected. Make sure I2C is enabled on the Raspberry Pi. In the terminal run `sudo raspi-config' and enable I2C in the "Interface Options". When everything went OK, you should now see that a device is detected on I2C address 0x77:

```shell
$ i2cdetect -y 1
    0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- 77
```

## Application

Because we are using JBang for this demo, all code is included in a single file.

First we need to start with an extra line to instruct JBang to handle it as a Java application, and download the dependencies.

```java
///usr/bin/env jbang "$0" "$@" ; exit $?

//DEPS org.slf4j:slf4j-api:1.7.35
//DEPS org.slf4j:slf4j-simple:1.7.35
//DEPS com.pi4j:pi4j-core:2.3.0
//DEPS com.pi4j:pi4j-plugin-raspberrypi:2.3.0
//DEPS com.pi4j:pi4j-plugin-linuxfs:2.3.0
```

Then we can define the imports as we would do in any Java file:

```java
import com.pi4j.Pi4J;
import com.pi4j.util.Console;
import com.pi4j.io.i2c.I2C;
import com.pi4j.io.i2c.I2CConfig;
import com.pi4j.io.i2c.I2CProvider;
import java.text.DecimalFormat;
```

The main method initializes the sensor, and loops 10 times to the process of resetting the sensor, reading the values, and pauzing.

```java
private static final Console console = new Console(); // Pi4J Logger helper

    private static final int I2C_BUS = 0x01;
    private static final int I2C_ADDRESS = 0x77; // When connecting SDO to GND = 0x76

    public static void main(String[] args) throws Exception {

        var pi4j = Pi4J.newAutoContext();

        // Initialize I2C
        console.println("Initializing the sensor via I2C");
        I2CProvider i2CProvider = pi4j.provider("linuxfs-i2c");
        I2CConfig i2cConfig = I2C.newConfigBuilder(pi4j)
                .id("BME280")
                .bus(I2C_BUS)
                .device(I2C_ADDRESS)
                .build();

        // Read values 10 times
        try (I2C bme280 = i2CProvider.create(i2cConfig)) {   
            for (int counter = 0; counter < 10; counter++) {
                console.println("**************************************");
                console.println("Reading values, loop " + (counter + 1));

                resetSensor(bme280);

                // The sensor needs some time to make the measurement
                Thread.sleep(100);

                getMeasurements(bme280);

                Thread.sleep(5000);
            }
        }

        pi4j.shutdown();

        console.println("**************************************");
        console.println("Finished");
    }
```

The reset and value readout is inspired by the Adafruit CircuitPython library and is [fully available in the sources on GitHub](https://github.com/Pi4J/pi4j-jbang/Pi4JTempHumPress.java). It's a combination of setting and reading registers, with different calculations. For example, for the temperature:

```java
byte[] buff = new byte[6];
device.readRegister(BMP280Declares.press_msb, buff);
long adc_T = (long) ((buff[3] & 0xFF) << 12) + (long) ((buff[4] & 0xFF) << 4) + (long) (buff[5] & 0xFF);

byte[] wrtReg = new byte[1];
wrtReg[0] = (byte) BMP280Declares.reg_dig_t1;

byte[] compVal = new byte[2];

DecimalFormat df = new DecimalFormat("0.###");

// Temperature
device.readRegister(wrtReg, compVal);
long dig_t1 = castOffSignInt(compVal);

device.readRegister(BMP280Declares.reg_dig_t2, compVal);
int dig_t2 = signedInt(compVal);

device.readRegister(BMP280Declares.reg_dig_t3, compVal);
int dig_t3 = signedInt(compVal);

double var1 = (((double) adc_T) / 16384.0 - ((double) dig_t1) / 1024.0) * ((double) dig_t2);
double var2 = ((((double) adc_T) / 131072.0 - ((double) dig_t1) / 8192.0) *
        (((double) adc_T) / 131072.0 - ((double) dig_t1) / 8192.0)) * ((double) dig_t3);
double t_fine = (int) (var1 + var2);
double temperature = (var1 + var2) / 5120.0;

console.println("Measure temperature: " + df.format(temperature) + "°C");
```

### Running the Application

Because JBang can download the dependencies and compile the code, we just need the Java-file to execute it:

```shell
$ jbang Pi4JTempHumPress.java
[jbang] Resolving dependencies...
[jbang]    org.slf4j:slf4j-api:1.7.35
[jbang]    org.slf4j:slf4j-simple:1.7.35
[jbang]    com.pi4j:pi4j-core:2.3.0
[jbang]    com.pi4j:pi4j-plugin-raspberrypi:2.3.0
[jbang]    com.pi4j:pi4j-plugin-pigpio:2.3.0
[jbang]    com.pi4j:pi4j-plugin-linuxfs:2.3.0
[jbang] Dependencies resolved
[jbang] Building jar...
[main] INFO com.pi4j.Pi4J - New auto context
[main] INFO com.pi4j.Pi4J - New context builder
[main] INFO com.pi4j.platform.impl.DefaultRuntimePlatforms - adding platform to managed platform map [id=raspberrypi; name=RaspberryPi Platform; priority=5; class=com.pi4j.plugin.raspberrypi.platform.RaspberryPiPlatform]
[main] INFO com.pi4j.util.Console - Initializing the sensor via I2C
[main] INFO com.pi4j.util.Console - **************************************
[main] INFO com.pi4j.util.Console - Reading values, loop 1
[main] INFO com.pi4j.util.Console - Measure temperature: 25.137°C
[main] INFO com.pi4j.util.Console - Humidity: 0.0%
[main] INFO com.pi4j.util.Console - Pressure: 101136.782 Pa
[main] INFO com.pi4j.util.Console - Pressure: 1.011 bar
[main] INFO com.pi4j.util.Console - Pressure: 0.998 atm
[main] INFO com.pi4j.util.Console - **************************************
[main] INFO com.pi4j.util.Console - Reading values, loop 2
...
[main] INFO com.pi4j.util.Console - **************************************
[main] INFO com.pi4j.util.Console - Finished
```

## Conclusion

Once again, JBang proves to be the perfect companion to experiment with Pi4J and an electronics component! 

## Read More 

The following sources have been used for this example:

* [Pi4J_V2-TemperatureSensor example code by Tom Aarts](https://github.com/Pi4J/pi4j-example-devices/blob/master/src/main/java/com/pi4j/devices/bmp280/README.md)
* [Bosch BMP280 Data Sheet](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bmp280-ds001.pdf)
* [Product page: "Adafruit BME280 I2C or SPI Temperature Humidity Pressure Sensor - STEMMA QT"](https://www.adafruit.com/product/2652)
* [Tutorial with Arduino code: "Adafruit BME280 Humidity + Barometric Pressure + Temperature Sensor Breakout"](https://learn.adafruit.com/adafruit-bme280-humidity-barometric-pressure-temperature-sensor-breakout/pinouts)
* [Adafruit_CircuitPython_BME280 library source code](https://github.com/adafruit/Adafruit_CircuitPython_BME280/blob/main/adafruit_bme280/basic.py#L248)