# LCM to ROS

This package is to subscribe to and republish [Lightweight Communications and Marshalling (LCM)](https://lcm-proj.github.io/) messages into [ROS](http://wiki.ros.org/) messages. LCM is an open-source messaging tool for high-bandwidth, low-latency communications based on UDP multicast.

The goal is to make a relatively generic system that requires only LCM message files and automatically generate republishers that subscribe to an LCM topic and republish the same messages onto a ROS topic.

This package automatically generates ROS message files, CPP files, a launch file and corresponding CMakeLists for catkin_make to create a set of republishers. The system has only been tested on Ubuntu 14.04 with ROS Indigo.

The user should only need to create the \*.lcm message files.

Note that this code is currently restricted to a limited number of data types. The LCM primitive list is available [here](https://lcm-proj.github.io/type_specification.html#type_specification_primitives). We have tested:

| LCM Primitive | Scalar | Fixed length array | Variable length array |
| --- | --- | --- | --- |
| int8_t | :white_check_mark: | :white_check_mark: | :white_check_mark: |
| int16_t | :white_check_mark: | :white_check_mark: | :white_check_mark: |
| int32_t | :white_check_mark: | :white_check_mark: | :white_check_mark: |
| int64_t | :white_check_mark: | :white_check_mark: | :white_check_mark: |
| float | :white_check_mark: | :white_check_mark: | :white_check_mark: |
| double | :white_check_mark: | :white_check_mark: | :white_check_mark: |
| string | n/a | n/a | :white_check_mark: |
| boolean | :white_check_mark: | :white_check_mark: | :white_check_mark: |
| byte | :white_check_mark:* | :white_check_mark:* | :white_check_mark:* |
*Note that bytes are treated as signed int8 by ROS

The LCM primitive types work fine because they can be directly converted into ROS types (really the types are standard data types). It is important to note that the code uses a reinterpret cast to directly refer to the incoming LCM message as a ROS message. This means that you should **NOT** directly edit the autogenerated msg file, because changing the order of items in the msg with respect to the LCM will mess things up. I'm also not 100% confident in my memory management, but it seems to work. The supplied example code shows an example of all the tested types listed above. ~~One remaining issue is that I do not have methods for using subclass messages (messages that contain other messages). I think this should be possible but I haven't done it yet.~~ Subclass messages work fine and an example is now included (the `simple_channel` message is included in the `new_t` type message in `example_t.lcm`).

A possibly better approach would be to try and read out the field names from the message and then set them directly, which might be safer than the pointer cast but uglier in terms of generating the CPP file, possible future work.


## Installation:

(Dependency) Install LCM:
```
git clone https://github.com/lcm-proj/lcm lcm
cd lcm
./bootstrap.sh
./configure
make
sudo make install
```

Clone this repository (assuming default catkin_make workspace):
```
cd ~/catkin_ws/src
git clone https://github.com/nrjl/lcm_to_ros.git
```

Add lcm messages to the 'lcm' folder. Some key notes about each lcm file:
*   Filename - the filename (without .lcm) is assumed to be the topic name for the republisher in the autogenerated code.
*   Struct name - The message type will be the `struct` name specified in the lcm file. The ROS message generated will reflect this.
*   Package name - The package name is set by the package specifier on the first line of the lcm file. lcm-gen will generate cpp headers in a folder called PACKAGE_NAME. The system should work automatically, but do not name two messages with the same structure name even with different package specifiers as they are lumped together by my catkin code.

So for the file called `example_t.lcm` in the lcm subdirectory that specifies an lcm message type `new_t` under the `exlcm` package, the system will generate the lcm cpp header file in `exlcm/new_t.hpp`, create a ros message in `msg/new_t.msg`, generate a simple republisher code in `autosrc/my_topic_republisher.cpp` that, when run as a ros node, subscribes to the lcm topic `example_t` with lcm message type `new_t` and republishes that message onto the ROS topic `\example_t` with ROS message type `new_t`. It will also add relevant lines to the CMakeLists in the autosrc directory and a line to the `all_republishers` launch file.

Also note that the repo ignores files (except the example) in the lcm directory, so that your message types will not be committed to the repo. You can specify your own package for the lcm messages, in which case they will be generated in a corresponding folder. 

Run the bash script and then make using catkin_make:
```
cd lcm_to_ros
./lcm_to_ros_generate.sh
cd ~/catkin_ws
catkin_make
```

This will parse the lcm subdirectory for .lcm files, and for each one attempt to:
    - create the lcm C++ header in the exlcm subdirectory (using `lcm-gen`)
    - create a corresponding ROS message (in the msg subdirectory)
    - create C++ republisher code (in the autosrc subdirectory)
    - add an entry to autosrc/CMakeLists.txt
    - add an entry to launch file launch/all_republishers.launch


Once complete, the publishers can be run with:
`roslaunch lcm_to_ros all_republishers.launch`
Note that this launch file places all republishers under the namespace `\lcm_to_ros`.

Each republisher can also be launched separately with:
`rosrun lcm_to_ros FNAME_republisher`
As usual, output topics can be remapped using standard ROS commands if required.

## Test example
This repo contains a test example. If you haven't added any other LCM messages (or even if you have) you should be able to confirm the code is working by performing the compilation steps, then:
* In a new terminal, run the ros core: `roscore`
* In a second terminal, run the example_t republisher: `rosrun lcm_to_ros example_t_republisher`
* In a third terminal: 
    * Check that the ROS topic */example_t* exists using: `rostopic list`
    * Then, show messages using: `rostopic echo /example_t`
* Finally, in a fourth terminal: `rosrun lcm_to_ros example_t_send_lcm`. This is a small LCM-only (no ROS dependencies) program that publishes a single message onto the LCM topic *example_t*
* Confirm that a message appears in the `rostopic echo` terminal.

## Rehash tool
I've also included a tool for overriding the [LCM fingerprint calculation](https://lcm-proj.github.io/type_specification.html). **This is not advised**, but can be handy when trying to force LCM message types to match for automated decoding. In general I suggest you don't use it, but if you need to, after creating a custom lcm message, add a line at the bottom of the lcm file that specifies a 64-bit fingerprint in this format (as in the example_t.lcm message, replace the 0x01... value with whatever you need it to be) `// HASH 0x0123456789abcdef`
The tool must be run for each message (or * glob of messages) you want to create an overridden hash version of:
```
./rehash-lcm-gen.sh lcm/example_t.lcm
```
This will create a new folder and hpp file with the package name specified in the lcm file appended with `_rehash`. You can proceed to use this message as usual, as shown in the example publisher, and it will force the LCM encoding and decoding to use the specified fingerprint value. The `-o` flag can be used if you also want to modify the autogenerated republishers to use the lcm message with the overridden hash values (must be used after `lcm_to_ros_generate` and must run `catkin_make` again to generate executables with overridden hash methods).
