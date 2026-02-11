# AeroShield_gnss_aviation_security_testbed
This repository contains the full implementation of a high-fidelity GNSS simulation-based cybersecurity testbed designed for aviation resilience research. The framework enables end-to-end signal generation, processing, and reception. In future, evaluation of realistic attacks like spoofing &amp; jamming will be evaluated. 

## System Setup ##
The entire system architecture is shown below. 

![SysArch3](https://github.com/user-attachments/assets/cdec9659-4728-4736-b07e-b0f1f1d2e417)

For the end-to-end simulation, it consists of 3 major sections: 
1. BlueSky:
BlueSky is an open-source air traffic simulator used to generate realistic aircraft trajectories that serve as the ground truth receiver position. These trajectories provide accurate latitude, longitude, altitude, and velocity data that define where the aircraft truly is, enabling downstream evaluation of GNSS positioning accuracy under clean and adversarial environments.

2. GPS-SDR-SIM:
GPS-SDR-SIM converts the BlueSky ground-truth trajectory into synthetic baseband/IQ GNSS signals, effectively acting as the satellite signal generator. It emulates how spacecraft would broadcast navigation signals to a receiver, allowing controlled spoofing, jamming, and signal distortion scenarios without physical satellites or RF hardware.

3. GNSS-SDR:
GNSS-SDR functions as a software-defined GNSS receiver that processes the IQ data produced by GPS-SDR-SIM to compute a navigation solution. It allows direct comparison between the receiver-derived position and the ground truth from BlueSky, making it possible to quantify accuracy, error behavior, and resilience under adversarial GNSS signal conditions.

### Initialization - Fedora OS ###
The end-to-end simulation has been implemented on Fedora Workstation 42. 
After installation of Fedora, check the desktop environment using this command

```$ echo $DESKTOP_SESSION```
-	The answer must be `GNOME` or `KDE` or `KDE:Plasma`

#### Anaconda ####
As a regular user:
Download Anaconda (Python 3.13 64-bit (x86) installer (please do not use the Miniconda)
  
```
$	chmod +x installer-package
  o	yes for all the options
  
$	source ~/.bashrc
```
Test installation
```
$	conda list

$	conda –version

$	conda create -n testenv python=3.11

$	conda activate testenv

$	python --version 
```
#### VSCode ####
1. Install the key and yum repository by running the following script:
```
$ sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc

$ echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\nautorefresh=1\ntype=rpm-md\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" | sudo tee /etc/yum.repos.d/vscode.repo > /dev/null
```
2. Then update the package cache and install the package using dnf (Fedora 22 and above):
```
$ dnf check-update
$ sudo dnf install code # or code-insiders
```
To check open VSCode &rarr terminal &rarr try to run a Linux command (sudo, conda, etc.)

### 1. BlueSky ###
This is the flight simulator. It is used to generate flight and its trajectories. It outputs csv files containing trajectory information to be fed to gps-sdr-sim (the next block). 

(You must have the newest Python 3.13.7)

Inside the VSCode
```
$ python -- version
```
-	Must be 3.13.7
```  
$ git clone https://github.com/kabartsjc/bluesky.git

$ pip install bluesky-simulator[full]
```
To test:
```
$ python3 BlueSky.py
```  
You must see, after a few seconds/minutes:

![bs1](https://github.com/user-attachments/assets/eab075c0-9b32-428c-8aef-ea79d8e47737)

![bs](https://github.com/user-attachments/assets/f42b4565-cbdd-464c-95b4-9a71210faf13)

We need an orchestrator file to create multiple epochs of the flight trajectory simulation. 
Open a new terminal (new tab if you already have an open terminal) in VSCode. 
Make sure you are in the directory: .....\bluesky or whatever the path. Make sure you have the following files in your current directory: Do ls and check for "BlueSky.py" and "orchestrator.py"

```
$ python3 orchestrator.py
```
This should generate a csv. The default scenario is created in ....\scenario\DEMO\demo-scenario.scn
New scenario can be created from bluesky which will be saved inside the DEMO directory. 
For modifying the simulation, this json file needs to be simulated "simu-config.json" (can be found in the root directory)

### 2. GPS-SDR-SIM ### 

For reference: https://github.com/osqzss/gps-sdr-sim

This unit takes input csv file as a dynamic input, or a static input for the flight trajectory can be provided. Read (https://github.com/osqzss/gps-sdr-sim) for additional instructions and information. 

Open a new terminal in Fedora.
```
$ git clone https://github.com/osqzss/gps-sdr-sim.git

$ gcc gpssim.c -lm -O3 -o gps-sdr-sim 
```
***
If missing packages, do
```
$ sudo dnf install gcc make glibc-devel
```
If your project needs FFTW (some GPS-SDR repos do), install:
```
$ sudo dnf install fftw-devel
```
If it needs git (to clone repos):
```
$ sudo dnf install git
```
***
```
$ ./gps-sdr-sim (for running the simulator)
```
***
If compilation fails: 
```
$  sudo dnf groupinstall "Development Tools"
```
***

But the above simulator needs to be configured with the following parameters:
```
Usage: gps-sdr-sim [options]
Options:
  -e <gps_nav>     RINEX navigation file for GPS ephemerides (required)
  -u <user_motion> User motion file in ECEF x, y, z format (dynamic mode)
  -x <user_motion> User motion file in lat, lon, height format (dynamic mode)
  -g <nmea_gga>    NMEA GGA stream (dynamic mode)
  -c <location>    ECEF X,Y,Z in meters (static mode) e.g. 3967283.15,1022538.18,4872414.48
  -l <location>    Lat,Lon,Hgt (static mode) e.g. 30.286502,120.032669,100
  -L <wnslf,dn,dtslf> User leap future event in GPS week number, day number, next leap second e.g. 2347,3,19
  -t <date,time>   Scenario start time YYYY/MM/DD,hh:mm:ss
  -T <date,time>   Overwrite TOC and TOE to scenario start time
  -d <duration>    Duration [sec] (dynamic mode max: 300 static mode max: 86400)
  -o <output>      I/Q sampling data file (default: gpssim.bin ; use - for stdout)
  -s <frequency>   Sampling frequency [Hz] (default: 2600000)
  -b <iq_bits>     I/Q data format [1/8/16] (default: 16)
  -i               Disable ionospheric delay for spacecraft scenario
  -p [fixed_gain]  Disable path loss and hold power level constant
  -v               Show details about simulated channels
```

Now, we need the GPS broadcast ephemeris file. Use the following link to extract the latest brdc file https://cddis.nasa.gov/archive/gnss/data/daily/. Make sure to register for free using your username and password. After you register, login and then use the link given to get the GNSS daily brdc file. The brdc file should match with the date of execution. So, if brdc files says brdc****.n file where the 4 digit number shows the day of the year e.g. 3020 is the 302nd day of the year. In the brdc****.n then represents the GPS satellite. Other variables represent other types of satellites like GLONASS, etc. 

Sample run:
```
$./gps-sdr-sim -e brdc3020.n -l 52.3 4.5 8453 -s 4000000 -b 8 -o output.bin
```
In the above sample run, read the configuration parameters given above. E.g. -e is used to provide the brdc file, -l to provide the static location of the aircraft, -s is the sampling frequency (note - use the same sampling frequency in the config file of the receiver, GNSS-SDR, that you will do in the next step). The above command should create an output file "output.bin" (feel free to modify the name of the output file, but make sure you use the path and the file name in the receiver block) containing the I/Q samples. These samples can be read & analyzed using GNU-radio. However, we will feed this output file to the GNSS-SDR (the receiver block). 

### 3. GNSS-SDR ###

(For reference: https://gnss-sdr.org/)

Open a new terminal in Fedora.
```
$ git clone https://github.com/gnss-sdr/gnss-sdr
```
***
If required:
If you are using Fedora 26 or above, the required software dependencies can be installed by doing
```
$ sudo yum install make automake gcc gcc-c++ kernel-devel cmake git boost-devel \
       boost-date-time boost-system boost-filesystem boost-thread boost-chrono \
       boost-serialization log4cpp-devel gnuradio-devel gr-osmosdr-devel \
       blas-devel lapack-devel matio-devel armadillo-devel gflags-devel \
       glog-devel openssl-devel libpcap-devel pugixml-devel python3-mako \
       protobuf-devel protobuf-compiler
```
In Fedora 36 and above, packages `spdlog-devel` and `fmt-devel` are also required.
*** 

Cloning the GNSS-SDR repository as in the line above will create a folder named gnss-sdr with the following structure:
```
 |-gnss-sdr
 |---cmake      <- CMake-related files.
 |---conf       <- Configuration files. Each file defines one particular receiver.
 |---docs       <- Contains documentation-related files.
 |---install    <- Executables will be placed here.
 |---src        <- Source code folder.
 |-----algorithms  <- Signal processing blocks.
 |-----core     <- Control plane, interfaces, systems' parameters.
 |-----main     <- Main function of the C++ program.
 |---tests      <- QA code.
 |---utils      <- some utilities (e.g. Matlab scripts).
```
```
$ sudo dnf install gmp-devel
```

Build and install GNSS-SDR
Go to GNSS-SDR's build directory:
```
$ cd gnss-sdr
```
Configure and build the application:
```
$ cmake -S . -B build (Fedora needs the CMake package installed first: sudo dnf install cmake)
$ cmake --build build
```
By default, CMake will build the Release version, meaning that the compiler will generate a fast, optimized executable. This is the recommended build type when using an RF front-end and you need to attain real-time. If working with a file (and thus without real-time constraints), you may want to obtain more information about the internals of the receiver, as well as more fine-grained logging. This can be done by building the Debug version, by doing:
```
$ cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug
$ cmake --build build-debug
```
This will create four executables at gnss-sdr/install, namely gnss-sdr, run_tests, front-end-cal and volk_gnsssdr_profile. You can run them from that folder, but if you prefer to install gnss-sdr on your system and have it available anywhere else, do:
```
$ sudo cmake --install build
```
This will also make a copy of the conf/ folder into /usr/local/share/gnss-sdr/conf for your reference. We suggest creating a working directory at your preferred location and store your own configuration and data files there.

With GNSS-SDR, you can define your own receiver, work with captured raw data or from an RF front-end, dump into files intermediate signals, or tune every single algorithm used in the signal processing. All the configuration is done in a single file. Those configuration files reside at the gnss-sdr/conf/ folder (or at /usr/local/share/gnss-sdr/conf if you installed the program). By default, the executable gnss-sdr will read the configuration available at gnss-sdr/conf/gnss-sdr.conf (or at (usr/local/share/gnss-sdr/conf/default.conf if you installed the program). You can edit that file to fit your needs, or even better, define a new my_receiver.conf file with your own configuration. 

Make sure to include the path of the the created `output.bin` file from gps-sdr-sim inside the config file `my_receiver.conf` to feed the generated I/Q samples to GNSS-SDR receiver. 

This new receiver can be generated by invoking gnss-sdr with the --config_file flag pointing to your configuration file:
```
$ cd ~/gnss-sdr/conf
$ gnss-sdr --config_file=my_receiver.conf
```
If the above steps doesn't work, if there is any issue or you need to see the expected output, use the following reference: https://gnss-sdr.org/my-first-fix/

## Attack Module ##
If the above sequence of scripts work properly, you have just implemented an end-to-end GNSS pipeline with a regular (un-jammed & un-spoofed) signal. Now, to introduce jamming & spoofing, the following sections will explain the details. 
### Jamming Implementation
Jammer transmits strong interference signal on the same frequency band, causing the receiver to lose the desired signal
- Complex baseband signal, 𝑥[𝑛]=𝐼[𝑛]+𝑗𝑄[𝑛]
- Received Jammed-Signal, 𝑦[𝑛]=𝑥[𝑛]+𝑤[𝑛], where, 𝑤[𝑛]  ~ 𝐶𝑁(0,2𝜎^2)
- Jammer strength, 𝜎= 𝛼∗𝑥_𝑅𝑀𝑆,      𝑥_𝑅𝑀𝑆= √(1/𝑁  ∑_(𝑛=0)^(𝑁−1)▒〖𝑥^2 [𝑛]〗) 


### Spoofing Implementation
