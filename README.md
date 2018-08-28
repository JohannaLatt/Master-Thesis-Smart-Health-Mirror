# Smart-Health-Mirror

This is the code base for the master thesis "Smart Health Mirror - Development of a Framework for Automated Evaluation of Physical Exercises" written by Johanna Latt at the [Hasso Plattner Institute](https://hpi.de/) in Potsdam, Germany, and in cooperation with the [Mt Sinai Institute for Next Generation Healthcare](http://www.nextgenhealthcare.org/) in New York City, USA.

# Installation

The code is split up into three main modules: The server, the mirror and the Kinect. Each module is in a separate repository. To clone the whole code, first clone this repository:

```
git clone https://github.com/JohannaLatt/Master-Thesis-Smart-Health-Mirror.git
```

Then, clone the submodules (otherwise the folders will be empty)

```
git submodule update --init --recursive
```

The submodules are private so please reach out to me for access!


## Description and Architecture

The framework consists of four independently running services. Each services can be run on separate machines to improve performance. 
The services are the following:

### <a name="Kinect"></a>Kinect (or other tracking system)
This service is responsible for providing 3D skeletal joint data. The framework is currently optmized to be run with a Microsoft Kinect v2. It includes a simple C#-program that streams the Kinect 3D data to the [server](#Server) via the [messaging-service](#Messaging). To use other tracking systems, a similar plugin would have to be written that is compatible with the [messaging-service](#Messaging) and that sends the data to the server. Ideally the data is sent in the format currently used throughout the serve infrastructure:

```
{
  'SpineBase': [331.2435, -419.485077, 2150.36621], 
  'SpineMid': [313.7696, -185.470459, 1992.33936], 
  'Neck': [294.341644, 44.1935768, 1821.40552], 
  'Head': [301.3063, 171.161179, 1819.05847], ...
 }
```

This format equals the joints provided by the Kinect v2. The [server](#Server) can also handle other formats, however, additional preprocessing will be required on the server-side.

Furthermore, the framework includes a python program that can simulate a Kinect in case no real Kinect is available. The simulation program already includes some samples, but the logging-data created by the [server](#Server) can also be directly fed into the simulator. The simulator can be started and paused using `t` (track) and `p` (pause). The file to be used can be specified with the `-f`-flag (specify the whole name of the file including the extension).

### <a name="Server"></a>Server
The server is the core of the application. It receives incoming data from the [kinect](#Kinect) and preprocesses that data into the expected format which consists of a bone- and a joint-mapping.

```
Joints:
{
  'SpineBase': [331.2435, -419.485077, 2150.36621], 
  'SpineMid': [313.7696, -185.470459, 1992.33936], 
  'Neck': [294.341644, 44.1935768, 1821.40552], 
  'Head': [301.3063, 171.161179, 1819.05847], ...
 }
 
Bones:
{
  'ClavicleRight': ['SpineShoulder', 'ShoulderRight'],
  'ShinLeft': ['KneeLeft', 'AnkleLeft'], ...
}
``` 

After the preprocessing, the main modules of the server are triggered and they all work on the preprocessed data, i.e. expect the format specified above. Each module does their own independent processing of that data and can then either trigger further internal actions or communicate with the [mirror](#Mirror).

The modules the server consists of can be specified in a config-file. At startup, the server starts one thread per module and each module receives its own message-queue. Incoming external and internal messages are distributed to all modules via their own message queues so that they can handle the incoming data in their own time without potentially impairing other modules that might be more performant.

Each module has multiple triggers that it can use to do its respective tasks. Currently these triggers are:
* External Triggers (normally used by preprocessing modules)
  * User detected by tracking device
  * New skeletal tracking data available
  * Tracking device lost tracking 
  * Mirror connected
* Internal Triggers (normally used by main modules and triggerd by preprocessing or main modules)
  * New preprocessed skeletal data is available
  * User changed exercise status (not exercising vs doing specific exercise)
  * User switched exercise stage (e.g. switching from going down to going up during a squat)
  * User finished a repetition of an exercise
  
 Example modules already inluced in the framework are:
 * Kinect preprocessing-module
   * Trigger: New skeletal tracking data available
   * Action: Turns incoming data into the server-wide format outlined above. In this case the conversion is simple since the incoming data is already in the correct format.
 * Welcome-module: 
   * Trigger: Mirror connected
   * Action: Displays a simple welcome action on the mirror whenever a mirror connects to the device
 * Logging-module:
   * Trigger: New preprocessed skeletal data is available
   * Action: Logs the new data into a log-file. The resulting log-file can directly be fed into the streaming simulator as mentioned [here](#Kinect). 
 * Render Skeleton-module
   * Trigger: New preprocessed skeletal data is available
   * Action: Sends the preprocessed data to the mirror so that the mirror can render the user's skeleton
 * Recognize Squat-module
   * Trigger: New preprocessed skeletal data is available
   * Action: Analyses the user's skeleton to tell whether the user is currently doing a squat, whether he is going down or up and whether he finished a repetition. This module triggers exercise-related internal triggers.
 * Evaluate Squat-module
   * Trigger: New preprocessed skeletal data is available & User finished a repetition of an exercise
   * Action: Evaluates a squat and gives feedback to the user by sending messages to the mirror. It evaluates whether the user keeps his head straight, whether the user keeps his shoulders straight and whether the user goes low enough during the squat. This module is highly customizable through the config-file.
 * Render Spine Graph-module
   * Trigger: New preprocessed skeletal data is available
   * Action: Takes the data of a specific, configurable joint (currently the SpineBase-joint) and sends that data to the mirror to visualize the movement in x, y and z during a squat.
   
All user-related information (joints and bones as well as exercise status) is also available in a User-object that is passed on through all modules. Some modules alter that object (in a threadsafe way) and other modules read from it. This avoids sending data between modules. Instead, this object capsulates all the data needed by all modules. 

### <a name="Mirror"></a>Mirror
The mirror is the GUI of the framework. It is built using the [kivy](https://kivy.org/)-framework to optimize for 2D-/3D-graphical rendering especially relevant for rendering the skeleton, but also for rendering dynamic text and graphs. The mirror only sends a message on startup to notify the server that it is ready. After that, it only receives data from the server. Currently, the mirror supports multiple operations that simplify communication between server-modules and the mirror:

* **Show Text**: The mirror can show text either statically at a certain position of the screen (and with a certain alignment, font-size and -color etc.) or dynamically following a user's joint over time. The text can also be faded in and out and it can be specified for how long the text stays.
* **Clear skeleton**: Clears the rendering of the skeleton off the screen. Used when tracking is lost.
* **Render skeleton**: Accepts the preprocessed skeleton format described under [server](#Server) and renders an OpenGL 2D skeleton on the screen. The skeleton is always positioned at the bottom of the screen to account for the assumption that the prototype is run on a full-room height mirror/screen, i.e. the rendered skeleton should 'stand' at the bottom of the screen. 
* **Change skeleton color**: Takes the name of a joint or a bone and changes the color of it respectively.
* **Update graphs**: Takes three values and displays them on the three graphs on the GUI, assuming the values are x-, y- and z-coordinates of a joint.

### <a name="Messaging"></a>Messaging
The messaging-service currently used is [RabbitMQ](https://www.rabbitmq.com/). Make sure to start the messaging service before running any other module, and adapt the messaging-service-ip in the config-files of all other modules. After that, no further action is required. The messaging-module handles the communication between Kinect and Server (omnidirectional) and Server and Mirror (bidirectional).


## Setup and Installation

### Messaging / RabbitMQ
Please follow the installation instructions [here](https://www.rabbitmq.com/download.html).

Under Windows, the procedure is as follows:
1. Install Erlang: You might have to use an old Erlang installation since the new one can lead to problems. If that is the case, use version 19.3 from [here](https://www.erlang-solutions.com/resources/download.html).
2. Run the Service
  * Run the console as admin
  * Go to the sbin-folder of the RabbitMQ installation 
  * Run `rabbitmq-server start`
  * You should see something like this:
  ```
   ##  ##
  ##  ##      RabbitMQ 3.7.6. Copyright (C) 2007-2018 Pivotal Software, Inc.
  ##########  Licensed under the MPL.  See http://www.rabbitmq.com/
  ######  ##
  ##########  Logs: C:/Users/.../AppData/Roaming/RabbitMQ/log/RABBIT~1.LOG
                    C:/Users/.../AppData/Roaming/RabbitMQ/log/rabbit@....log

              Starting broker...
 completed with 3 plugins.
 ```
3. To test whether the server is running correctly execute `rabbitmqctl status` (again in the sbin folder)
  * If the server is running correctly, you should see a message starting with `Status of node 'rabbit@xxx' ...`
  * If everything worked, the server is running on port 5672 on your computer
4. The Kinect C# program needs to be able to access our RabbitMQ service, for that we have to configure an additional user for our messaging-service. The new user will have the username `Kinect` and password `kinect`
  * Enable the management console by running `rabbitmq-plugins enable rabbitmq_management`
  * Now you should be able to go to (http://localhost:15672/) in your browser and see the management GUI of RabbitMQ
  * The default username and password is `guest`
  * Go to the Admin-tab and click on „Add a user“ at the bottom, set the name to 'Kinect' and the password to 'Kinect', set the user as admin and confirm
  * Click on the name of the new user to open more settings
  * Click on „Set permission“ without changing anything to allow access
5. After this, the KinectStreaming.exe should be able to connect and you can go ahead and start the other services


### <a name="VirtualEnv"></a> Python Virtual Environment 
To run the python programs included in this framework, it is recommended to use virtual environments to handle your dependencies. If you want to use virtual environments, install the necessary packages like this:

1. First, make sure to have *Python 3.6* installed on your system.

2. Then, install `virtualenvwrapper`
  * Mac/Linux: `pip install virtualenvwrapper`
  * Windows: `pip install virtualenvwrapper-win`

3. Change the environment variable `WORKON_HOME` if you dont want to use the default location for the virtual environments.
4. After that, create the virtual environment that you will be using for this project (either all services at once or one environment per service if you use multiple machines):
```
mkvirtualenv [name]
```
5. Activate the environment on the console you want to run the server, the simulator or the mirror from: 
```
workon [name]
```
6. Finally, install the required packages from the `requirements.txt` file of the respective component: 
```
pip install -r [link to requirements.txt]
```

### Kinect
To run the actual Kinect, connect it to your Windows computer after having installed the [Windows Kinect v2 SDK] (https://www.microsoft.com/en-us/download/details.aspx?id=44561). Make sure it is recognized. Then just double-click [KinectStreaming.exe](https://github.com/JohannaLatt/Smart-Health-Mirror/blob/master/Kinect/Windows/KinectStreaming/bin/Release/KinectStreaming.exe). The command line prompt that opens expects you to enter the IP address of the RabbitMQ server (without the port). Enter the IP address (or just enter `localhost`) and press enter. The program should then automatically connect to the messaging-service and to the Kinect and start streaming data (which will also be displayed in the prompt).

### Kinect Simulator
Alternatively to using an actual tracking device, the simulator can also be used:

1. Make sure to have the necessary requirements installed: `pip install -r [link to requirements.txt in Simulator folder]` (if you use virtual environments, make sure to activate it)
2. Go to the Kinect Simulator-folder in this repository: `cd Kinect/Simulator`
3. Update the RabbitMQ-messaging-server-ip in the [config-file](https://github.com/JohannaLatt/Smart-Health-Mirror/blob/master/Kinect/config/kinect_config.ini) if needed
4. Start the simulator
```
python index.py -s [stanford, cornell] -f [filename including extension]
``` 
5. Enter `t` and press enter to start the tracking simulation, `p` stops the simulation and `q` quits the simulator
6. To use data recorded with this framework as the basis for the simulator, take the log-file created by the server (it will be under /Server/Server/logs) and copy it into /Kinect/data/sample-kinect. Let's assume the file is named `tracking-data.log`. To use this data with the simulator, start it with: `python index.py -f tracking-data.log`. 

### Server
The server is a simply Python flask server that we need to start:

1. Make sure to have the necessary requirements installed: `pip install -r [link to requirements.txt in Server folder]` (if you use virtual environments, make sure to activate it)
2. Go to /Server/Server in this repository: `cd Server/Server`
3. Set the `FLASK_APP`-environment variable to \_\_index\_\_.py
  * Mac/Linux: `export FLASK_APP= __index__.py`
  * Windows: `set FLASK_APP= __index__.py`
4. Update the RabbitMQ-messaging-server-ip in the [config-file](https://github.com/JohannaLatt/Smart-Health-Mirror/blob/master/Server/Server/config/mirror_config.ini) if needed
5. Run the server:
```
flask run
```

### Mirror
The mirror is a kivy appliation that is run as a normal python application:

1. Make sure to have the necessary requirements installed: `pip install -r [link to requirements.txt in Mirror folder]` (if you use virtual environments, make sure to activate it)
  * For garden (used for the graphs) an additional requirement has to be manually installed: `garden install --upgrade graph`
2. Go to /Mirror in this repository: `cd Mirror`
4. Update the RabbitMQ-messaging-server-ip in the [config-file](https://github.com/JohannaLatt/Smart-Health-Mirror/blob/master/Mirror/config/mirror_config.ini) if needed
5. Run the application
```
python index.py
```





