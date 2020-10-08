## System Architecture

The System is divided into two components, a sensor array connected to a main computing board and a ROS package on a host PC

### Sensor array device:

![ ](https://raw.githubusercontent.com/fontysrobotics/Human-Body-Tracker-v02/master/docs/res/device_system.png  "Diagram 1: Sensor array device")

**Note:** The figure represents the basic architecture of the on body device, currently the project is in research phase and the system is to be expanded and modified as the project enters development phase.

The System revolves around a System on Chip (SOC), for the project a XIlinx Zynq 70007S on a support board. The choice for this board and chipset was made in order to allow the research into the efficiency of offloading the bulk of sensor processing in the FPGA, leaving the CPU to interface with the rest of the high level modules. The FPGA offers the advantage of strict timing and parallel processing of the raw sensor data streams that the single core CPU can't achieve running an operating system. The disadvantage appears in the form of the lower frequency the FPGA runs relatively to the CPU and the higher difficulty to develop the solution. Depending on the results of the research of the development phase, the final version of the project will either process the sensor in the CPU, FPGA or a hybrid of both.

Taking a look at the SOC - FPGA we can see the following blocks in the the design:

- **SPI (optional I2C):** The devkit contains a class of modules dedicated for high efficiency SPI and I2C communication with support for various modes, this offloads the need to spend time developing the FPGA blocks and it ensures a high quality and reduction of errors relatively to self developed modules. The block acts only as hardware decode/encode interface for the rest of the system, multiple parallel blocks can be used for higher data throughput. Most of the sensor on the market run SPI Standard in Slave Mode or I2C Data Bus. The reason why SPI is a good choice over I2C is that the transmission is more reliable when the length of the cables increase and bus contains both an input and output data lane. I2C advantage is the smaller amount of wires used in the bus connection as well as the ease of use in adding new device to the bus. The project will prefer the SPI interface for sensor that supports both and will use I2C when the need arises.

- **Stage I Sensor processor:** The block has a dual role, to process sensor data and to execute calls to the sensor array in order to calibrate, initialize and request data. The block will translate simple instructions from the CPU and encode the instructions in a sensor compatible format, pass them to the sensor array and decode and push any data or responses the sensors return. The block can be discarded depending on the data the sensor outputs and the rate of the calls.

- **Multiplexer:** The block acts like a switch between the AXI Bus and the rest of the system, its role is to either push all the instructions to the Sensor Processing block or to allow a direct connection to the SPI blocks array. Its configuration is reflected to the higher levels in order to load the proper drivers

At the higher level we have a standard ARM Single Core CPU with attached memory and storage blocks. The CPU runs a Linux kernel and Ubuntu System without a graphical session and with a SSH and debug port interface enabled. The CPU has access to the USB controller on board where an WiFI modules is plugged, offering to the board wireless capabilities. The OS image is containing a collections of important processes, among them the following being of interest to this project:

- **Optional Kernel Drivers:** The system runs a stripped down kernel, containing only the core modules required to run the system with optional drivers loaded on startup. All the drivers are in the standard tree, ensuring tight coupling with the rest of the kernel. Depending on the multiplexer configuration the driver loaded differs.
	- *In sensor processing mode:* The loaded driver is Generic UIO (User Input/Output) is loaded, this driver allocates a chunk of registry memory and synchronize it with the FPGA side, the memory is split in sectors for writing and reading blocks. The data transfer is done by opening this registers and writing and reading to them as standard streams.
	- *Pass trough:* The loaded driver is AXI-SPI/AXI-I2C Driver, this allow the kernel to interface directly with the sensor array for data exchange and offer an specialized interface for the rest of the system. This specialized interface allow faster data transfer between FPGA and CPU but place all the data processing on the high level program.

- **Sensor Data Processor:** A python program accessing the kernel driver for exchanging data with the FPGA. Depending on the sensor selection and the multiplexer mode, the block will do at the minimum data packing in and issue simple instructions or it will communicate with sensor array, format the data and process it into usable values.

- **Data Packet Manager:** A sub component of the the python program, it is a lightweight server that allow remote procedures calls and data transmission between the board and the rest of the system. The server is reactive, it won't send data without the client system requesting it.

**Note:** The sensor processing blocks, either in the CPU and FPGA, need to have a final output of corrected joints positions, that is a quaternion representing the drift correct position of the sensor in 3D space. The system will be a mix of well known filtering algorithms and conditional operators to deal with large errors.

The rest of the diagram is represented by hardware links:

- **Carrier Board:** The board were the SoC and the rest of the associated components is placed as well contains the exposed pins for the sensor ports as well as various sockets for the USB WiFi, power plug and debug port. The board is an Avnet Minized in Flash Boot Mode, the emmc loaded with the Ubuntu system and Linux Kernel and the QSPI memory with the boot loader and the FPGA configuration.

- **Battery Block:** A simple Li-ion battery pack with a 5V, 1A mini usb connector for the board power socket

- **Sensor Array:** The sensors type and positioning is described in a dedicated document. It is important to note that all the sensors in an array are sharing the power lines. For SPI the MOSI/MISO lines are connected to all sensor and different Slave Selects pins are used for each sensor. For I2C the SDA and SCL lines are linked to all the sensor in the array, testing is required to decide if signal boosters are needed.


## ROS package

![ ](https://raw.githubusercontent.com/fontysrobotics/Human-Body-Tracker-v02/master/docs/res/client_system.png  "Diagram 2: Ros System")

**Note:** In the figure it can be seen the core nodes or modules that are to be used for the project, all the modules are linked together through the ROS communication infrastructure. The figure contains only the topic names, for the datatypes reference the topic description

The package revolves around two main components:

- **Transform Frame Management Node:** The module is part of the standard ROS collection. Using a reference frame, joints and links the node creates a tree view representing the position of various components in the 3D space, a timestamp is used to keep track of the changes in time. Navigating through the tree is possible to calculate through the module the relative position and angle of two selected components. Alternatively changing a single component position the rest of the elements in the tree adjust accordingly. Looking at the elements used by the node we have the following:
	- *Reference Frame:* The reference frame is a plane representing the ground truth, it offers to the whole tree a point where everything is connected. The reference frame is generally represented by the ground floor and this project will use this as its reference
	- *Joint:* it represent a point in the tree where an angle is given in the shape of a quaternion, this mathematical representation is preferred as it doesn't present the issue of Gimbal Locks. When the angle changes any forward element in the tree, that is any element connected from the joint to the endpoint, rotates as full assembly. For the purpose of this project the joints are represented by the various joints present in the human body
	- *Link:* it represents a distance between two joints or a joint and an endpoint, the links are perfectly rigid and the only deformation they can have is a change in its length. For the purpose of this project the links are keeping a strict length, that is they never change during the run of the program as they represent elements of the human bone structure.
	
- **Interface Node:** The module is part of the provided package in this repository. The purpose of the node is to connect wireless to the sensor device and exchange data in a compact format in order to increase the throughput of data on the connection and reduce computation time on the sensor device. When the sensor array sends processed sensor data, the interface node will decode it and update the joints through the TF system, effectively updating the virtual representation of the human body. If the sensor sends any status or error updates, the node will push it through the "/status" topic to any nodes listing on it and to the visualization software. Additionally the status change is timestamped and saved to a file. The node listens to a "/cmd_interface" topic for any configuration changes it should push to the sensor device

Beside the core components of the project, secondary components are present:

- **Model Storage:** The TF system makes use of lines and points in order to represent the changes, this allow a concise presentation of the data in a visual mode but it can be less pleasing to the eye. Making use of 3D models broken down in various components, we can overlay over the frames more visually pleasing elements. the Elements position is updated based on the TF changes. Generally speaking this model storage is named a "robot model", for the purpose of this project it can be considered it is an "human model"
- **Parameter Server:** It is a server running as part of the ROS main system, it is a database where variables are mapped with values during the start up of the system, the nodes running on the ROS framework can request configuration information stored in this database of edit values in order to change other nodes. For this system, a file is provided with configuration for the interface node on how to connect to the sensor array device, what constant adjustments to make to the sensor data and what configuration to push to the sensor device during the startup sequence.
- **Rviz:** Is a GUI application and is part of the ROS collection. The application offers generic visualization functionality, one of the options is the visualization of the TF tree in 3D space with the "human model" overlay. This functionality satisfy the project requirements but for future versions a custom dedicated visualization tool should be used as Rviz can't push commands by itself to the system and can be used only on systems with ROS installed.
