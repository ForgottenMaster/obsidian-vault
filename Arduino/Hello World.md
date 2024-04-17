This is the first in hopefully a long series of posts in Arduino development as I learn about embedded development, microcontrollers, and electrical engineering.

---
# The hardware #

The target hardware for the experimentation will be this very Arduino Uno

![Arduino Uno Front](Arduino%20Front.jpg)

And, a quick look at the back will tell us that this is the R3 model

![Arduino Uno Back](Arduino%20Back.jpg)

However I believe any Arduino Uno model, as well as a few other Arduinos will work with the methods and APIs we're using

The second component we will need is the cable to connect the Arduino to the computer, this is a male USB-A (plugs into the computer) to USB-B (plugs into Arduino) cable

![Arduino Cable](Arduino%20Cable.jpg)

This will be all that is needed to set up the environment and a "Hello, World!" application, but obviously as we go through different projects, etc. we'll require more peripherals.

---
# The coding environment #

We will be using Rust for all Arduino development which has the ability to compile to various embedded target platforms.

More generically, Rust has the ability to compile for any target platform for which LLVM supports, and a linker is available for.

Rust is a systems programming language rivalling C and due to the language itself being more expressive (such as expressing ownership), has a very good optimising compiler.

The Rust environment is easy to install, simply follow the directions provided on the [Rustup](https://rustup.rs/) website which will install Cargo and the Rust compiler.

#### The repository ####

All of the code for this Arduino experimentation will be put in the [Arduino Uno](https://github.com/ForgottenMaster/arduino-uno) repository.

This repository is setup with GitHub workflows to check compilation and formatting on code submission, similar to my other repositories, except I've removed code coverage and testing workflows.

#### Dependencies ####

There are a few dependencies that need to be set up on the development machine to support Arduino development. These are detailed below.

###### AVR ######
As mentioned previously, the Rust compiler is able to compile to any platform that LLVM supports, however requires an appropriate linker to be present on the system.

In the case of AVR development, we will need to make sure we install the tools for an AVR development environment.

On Linux, these can be installed with the following command

```
sudo apt install avr-libc gcc-avr pkg-config avrdude
```

On windows, the process is a little more involved, but not exactly difficult.

1. Install the WinAVR package, which sets up an AVR development environment. The WinAVR package can be found [HERE](https://sourceforge.net/projects/winavr/files/)
2. WinAVR is a little out of date, in particular the version of avrdude is too old for us to use with the library we're going to be using. Download the latest version of avrdude from [HERE](https://github.com/mariusgreuel/avrdude/releases) and replace the **avrdude.exe** binary with it

###### Ravedude ######
Ravedude is a program which handles flashing the compiled program onto the device. With Ravedude installed, running

```
cargo run
```

in a project that is setup to compile AVR executables will result in the executable being built, and then pushed to a connected device.

To set this up on either Windows or Linux (or any other OS), it can be installed through Cargo with the command

```
cargo +stable install ravedude
```

---
# Creating the project #

The API we will be using is called [avr-hal](https://github.com/Rahix/avr-hal) which stands for **AVR Hardware Abstraction Layer**.

A HAL will handle mapping coding structures (such as an enum of pins) to the actual pins on a device. It's intended that a HAL will abstract the hardware from the coder, leaving them to focus only on what they want it to do.

The author provides a neat template that can be used with **cargo-generate** to create the project.

To create a project with this method, simply install the cargo-generate program, conveniently through Cargo itself:

```
cargo install cargo-generate
```

Secondly invoke it as such:

```
cargo generate --git https://github.com/Rahix/avr-hal-template.git
```

It will ask for the project name, and the target Arduino device (I chose Arduino Uno).

---
# Running the project #

Thanks to the HAL, and Ravedude, it's easy to build and run the generated project.

Firstly, plug the Arduino device into the computer.

Secondly, run the following command

```
cargo run --release
```

This will compile the sample program, and push the executable to the Arduino, resulting in a lovely little flashing light as shown in the video below

![[Demo.mp4]]