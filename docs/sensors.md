## Sensor Selection

This document contains the sensors that were considered for the project as well why some offer better functionality than the others and how that functionality impacts the project

#### Sparkfun Board for STMicroelectronics LSM9DS1
The board itself is just a carrier for the sensor and such alternatives with the same sensor can be used. The board allows robust access to the connections of the sensor, no specialized connector is required. While the size of the board can create issues in more compact designs, the chip can be used on custom PCBs given the access to proper tools. The board and sensor combination can be found on the official website at approximately 16 euros.

Looking at the sensor itself we have the LSM9DS1 manufactured by STMicroelectronics, the datasheet can be found on the official [website](https://www.st.com/en/mems-and-sensors/lsm9ds1.html). The sensor contains a digital Accelerometer, Gyroscope and Magnetometer with direct access. The sensor allows direct communication through I2C and SPI with logic levels of 1.9 to 3.6V. The accelerometer and gyroscope data can be requested as 16 bits read through the communication interfaces and the sensitivity value is given by the range the sensors are set at.

The sensor use a simple communication scheme, the master device push a value to a register and the sensor respond. This allow it to be easy integrated in this design but due to the direct data access it is required for extra processing to be done in the FPGA component of the board or on high level software on the CPU. Additionally this raises the need for implementing error correction for drifts and malformed reads of the data which takes both processing power and adds latency to the processing pipeline. Both of this require specialized algorithms, one possible solution can be the use of the Madgwick algorithm, libraries supporting this algorithm can be found for various languages and for the CPU component.

Summary of the sensor:

- Accelerometer: range 2-16 g, sensitivity 0.061-0.732 mg/LSB and zero-g level offset +/-90mg
- Gyroscope: range 245-2000 dps, sensitivity of 8.75-70 mdps/LSB and zero-rate level offset 30dps
- Magnetometer: range 2-16 gauss, sensitivity 0.14-0.58 and zero-gauss level offset +/- 1 gauss
- Both SPI and I2C connections, no specialized protocol
- No on board processor or compensation algorithm
- Cheap choice for the project relatively to other choices

### Adafruit Board for Bosch BNO055
Similar with the previous sensor, the board acts as a holder of the chip and exposes the connections to pins. The Adafruit board contains some additional passive components as well as a special connector for a more rigid coupling in order to avoid detaching during board movements. The board is quite compact and considering the footprint of the connectors it might prove easy to integrate in a small design. The price of the board and sensor combination is 35 euros on the official shop. The sensor itself can be bought for custom designs.

The sensor on the board is a BNO055 by Bosch, the full datasheet provided by the manufacture company at [link](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bno055-ds000.pdf). The sensor contains a 3 axis, Accelerometer, Gyroscope and Magnetometer as well a temperature sensor. The sensor supports I2C when coupled with the board and UART when used by itself for custom designs, both 3.3 logic levels and 3V for interrupt pins. A note regarding the I2C, the sensor uses the address of 0x28 with an alternative address of 0x29 selected through a pin, this means that only two sensors can sit on a data bus and each use an interrupt pin.

In the same chip it is embedded an ARM M0 running a custom Bosch firmware. The firmware manages the power mods on the chip, can swap the chip in sleep and high performance mode as well as manages the data type. The chip contains an algorithm based on the Extended Kalman Filter and executes sensor fusion. The sensor fusion compensates for drifts and ensures that the noise is removed from the raw sensor values, all the computation is done efficiently on the sensor chipset and updates at a rate of 100Hz. The data can be exchanged through reading and writing to the registers through I2C, the chip returns in direct sensor access mode the raw values and in fusion mode, a equation representing the orientation of the sensor relatively the earth ground.

The communication is simple, all the data is accessed through read and write on 8bit registers, in some cases data is broken down over multiple registers. Due to the sensor fusion mode there is no need for extra processing to be done by the FPGA or the CPU beyond the configuration and data requests. Considering this and that only 2 devices can sit on a bus, in the FPGA it will be define an I2C connected directly to the CPU multiplexer for each pair. In order to facilitate faster response times, the interrupts from the sensors will be passed to the CPU interrupt block.

Summary of the sensor:

- Accelerometer: range 2-16g, zero-g offset +/-80mg, nonlinearity of 0.5
- Gyroscope: range 125-2000 degrees/s, zero-rate offset +/- 1 degree, nonlinearity of 0.05
- Magnetometer: range 1200-2300 μT, zero offset before calibration +/-40μT, zero offset after calibration +/-2μT
- I2C only for board, 2 devices per bus, no specialized protocol
- On board ARM M0, calibration, drift compensation and sensor fusion
- Outputs Quaternion and Euler angles in sensor fusion with update rate 100Hz
- Efficient option for price performance

### XSens MTi-3 AHRS
The sensor is composed from multiple chips and oscillating crystal on plastic chip carrier, square shaped with an approximate size of 12mm. The chip carrier can be soldered directly on a breakout board or placed into an PLCC-28 socket. Considering the socket options the project needs to create a PCB with the socket and the headers linked to the pins. This allow easy custom designs with non specialized tools but add a timing overhead during the prototype phase. The chipset can be found on the official website and resellers for a price in the range 200-300 euros.

The sensor is part of the MTi-1 series by XSens, for the project the model MTi-3 is considered, the datasheet for the series can be found at [link](https://www.xsens.com/hubfs/Downloads/Manuals/MTi-1-series-datasheet.pdf). The sensor is composed from an 3 axis Accelerometer, Gyroscope and Magnetometer with an addition of a clocking module and temperature sensor. The sensor offer interfaces for SPI, I2C and UART at 3.3 logic levels with the note that the chipset can run only a single interface at the same time based on the select pins.

The MCU (microcontroller) on the chipset executes a multitude of stages defined as follows:

- **Sensor data read:** Using the proprietary Xsens strapdown algorithm, the MCU reads the sensors and applies compensation for drifts, spikes in motion and temperature changes at a rate of 800Hz. The data is used internally by the MCU in the MTi-3 version of the sensor
- **Sensor data fusion:** Inside the MCU is containing the proprietary Xsens Engine that combines the processed sensor data into a corrected quaternion value, the algorithm disables certain sensor data when it detects that it is unreliable, the output is updated at a frequency of 100Hz
- **Xbus data transfer** The MCU uses the proprietary XBus protocol, the protocol definition can be found on the official website. Based on its description, it ensures reliable and fast data transfer between the chipset an any other device in the network.

While the board offers a variety of interfaces at hardware level, the existence of specialized algorithms increases the difficulty of implementing the communication directly at the FPGA layer and it requires for the multiplexer to be set in pass through mode and let the Linux kernel and high level software handle the communication.

Summary of the sensor:

- Accelerometer: range of 16g, nonliterary 0.5
- Gyroscope: range 2000 degress/s, nonlinearity 0.1
- Magnetometer: 8 gauss, nonlinearity 0.5
- Various communication interfaces, use SPI for the project, it requires to implement XBus protocol
- On board MCU with specialized algorithm for data acquisition and processing to correct drift and errors
- Corrected sensor data required at 800Hz and processed pose updated at 100Hz, pose supports euler angles and quaterions
- High price compared with other options, the sensor is used in a successful application by the same company
- Requires socket in order to connect reliably

## Final Considerations
The differences in the raw values based on the datasheets are small enough to consider the sensors equal in quality, the other factors are used in order to make the decision for the project:

- **Price:** The LSM9DS1 offers the most complete package for the lowest price among the considered sensor for raw sensor capabilities
- **Hardware board:** Both the LSM9DS1 and BNO055 are placed on carrier boards with simple generic headers, the BNO055 allowing the use of connectors that offer more reliable links, from an expansion point of view the MTi-3 design that makes use of the socket allow the development of custom boards without the need of specialized tools with the drawback that a board needs to be design in order to use the sensor chipset.
- **Connectivity:** from all the boards the BNO055 is the only sensor that doesn't support SPI and has the limitation that only two devices can be used by bus/endpoint in the FPGA, this issue impacts the design of the cabling and amount of endpoints in the FPGA; it is also important to consider that MTi-3 requires for the main board to implement its custom protocol
- **Processing Power:** the BNO055 and MTi-3 executes computation on the chipset at the frequency of 100Hz, this removes the sensor signal processing pipeline and reduce the load on the CPU and FPGA as well as remove the overall latency from data traveling from the sensor to the client laptop; for the LSM9DS1 the computation needs to be done in the FPGA or high level CPU program
