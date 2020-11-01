# Learn Rust part 1

## Introduction

I have been a professional developer for almost 20 years, and like many professional developers I have used many programming languages in my career (C, C++, Fortran, Java, Scala, Javascript, TypeScript, Python, Go, etc.).
It's been a while now that I've been wanting to learn Rust and write an article on Medium so I decide to do both at the same time.
Although it is not my mother tongue, I will write my articles in English. Maybe there will be some mistakes or some weird turns of phrase, please be kind but do not hesitate to correct me, I'm only asking for improvement :).

In this first article I describe how to create a complete Rust development environment, usable from a browser and contained in a docker image. We will also write our first program and learn how to use the debugger.


## Create a portable development environment

To begin we will create a Dockerfile containing all the necessary tools to program in rust. This is how this Dockerfile looks like:
```dockerfile
FROM ubuntu:20.04

# Upgrade and install curl, sudo and essential development tools
RUN apt update -y && apt upgrade -y && \
    apt install -y curl git sudo build-essential lldb

# Add a user `rustdev` so that you're not developing as the `root` user
# The user needs to sudoer be able to install code-server
RUN adduser --gecos '/usr/bin/bash' --disabled-password rustdev && \
  echo "rustdev ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/nopasswd
ENV SHELL bash

USER rustdev
WORKDIR /home/rustdev
# Create USER environment variable to prevent "could not determine the current user, please set $USER" error when running "cargo new ..."
ENV USER rustdev

# Install rustup
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs > rustup.sh
RUN sh rustup.sh -y && rm -f rustup.sh
ENV PATH /home/rustdev/.cargo/bin:$PATH

# Install code-server 
RUN curl -fsSL https://code-server.dev/install.sh > install.sh
RUN sh install.sh --method=standalone
RUN rm -f install.sh
RUN sudo chown -R rustdev:rustdev /home/rustdev/.local
ENV PATH /home/rustdev/.local/bin:$PATH

# Install code-server extensions

## Install extensions dependencies
RUN rustup component add rust-analysis --toolchain stable-x86_64-unknown-linux-gnu
RUN rustup component add rust-src --toolchain stable-x86_64-unknown-linux-gnu
RUN rustup component add rls --toolchain stable-x86_64-unknown-linux-gnu

## Adds language support for Rust
RUN code-server --install-extension rust-lang.rust

## Alternative Rust Language Server
RUN code-server --install-extension matklad.rust-analyzer
### Download and Install rus-analyzer binary
RUN mkdir -p /home/rustdev/.local/share/code-server/User/globalStorage/matklad.rust-analyzer/
RUN curl -L https://github.com/rust-analyzer/rust-analyzer/releases/latest/download/rust-analyzer-linux -o /home/rustdev/.local/share/code-server/User/globalStorage/matklad.rust-analyzer/rust-analyzer-linux
RUN chmod u+x /home/rustdev/.local/share/code-server/User/globalStorage/matklad.rust-analyzer/rust-analyzer-linux

## Debugging tools
## Install from the vsix instead of using extension id because of this issue: https://github.com/vadimcn/vscode-lldb/issues/314
RUN curl -L https://github.com/vadimcn/vscode-lldb/releases/download/v1.6.0/codelldb-x86_64-linux.vsix -o codelldb-x86_64-linux.vsix
RUN code-server --install-extension codelldb-x86_64-linux.vsix
RUN rm -f codelldb-x86_64-linux.vsix

## Toml extension, usefull to edit cargo configuration files
RUN code-server --install-extension bungcip.better-toml

# Remove sudo package
USER root
ENV SUDO_FORCE_REMOVE yes
RUN apt -y remove sudo

# Run code server
USER rustdev

CMD ["code-server", "--bind-addr=0.0.0.0:8080", "--auth=none"]     
```

Although it's not extremely complex it took me longer than I expected to create this Dockerfile and I think it's interesting that we go through it together. 

The first part is very common I choose ubuntu:20.04 as base image because ubuntu is widely adopted by the docker community thus it is easy to find support and the version 20.04 because it is a LTS. The RUN directive installs some necessary system tools:

- [Curl](https://curl.haxx.se/) to download some installation scripts and package
- [Git](https://git-scm.com/) because it is used by [Cargo](https://doc.rust-lang.org/book/ch01-03-hello-cargo.html) the Rust build system and package manager
- sudo because it is needed to install [code-server](https://github.com/cdr/code-server) a very nice [VS Code](https://github.com/Microsoft/vscode) clone with a web UI
- [build-essential](https://packages.ubuntu.com/focal/build-essential) a meta package containing essential building tools
- [LLDB](https://lldb.llvm.org/) the debugger of the [LLVM](https://llvm.org/) project

```dockerfile
FROM ubuntu:20.04

# Upgrade and install curl, sudo and essential development tools
RUN apt update -y && apt upgrade -y && \
    apt install -y curl git sudo build-essential lldb
```
As it is a bad practice to create root containers I create a user named *rustdev*. The default SHELL for this user is BASH, the home directory is  */home/rustdev* , the password is disabled. To be able to install *code-server* it is also needed to add *rustdev* to the sudoer list to allow rustdev to use sudo without password. To be perfectly honest, giving to a user the possibility to run sudo without password is also not a very good practice. This will be fix later by removing the package sudo. The SHELL environment variable is also set to make *bash* the default shell.

```dockerfile
# Add a user `rustdev` so that you're not developing as the `root` user
# The user needs to sudoer be able to install code-server
RUN adduser --gecos '/usr/bin/bash' --disabled-password rustdev && \
  echo "rustdev ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/nopasswd
ENV SHELL bash
```

The next section set the user to be *rustdev*, the working directory to be */home/rustdev*, the home directory of *rustdev* user. The USER environment variable is defined to be able to use Cargo later. If you do not define the USER environment variable running *cargo new ...* int the container will raise the error: *"could not determine the current user, please set $USER"*

```dockerfile
USER rustdev
WORKDIR /home/rustdev
# Create USER environment variable to prevent "could not determine the current user, please set $USER" error when running "cargo new ..."
ENV USER rustdev
```

Now its time to install [*rustup*](https://rust-lang.github.io/rustup/) the Rust toolchain installer. The script *rustup.sh* installs executables  like *rustc* and *cargo* in *~/.cargo/bin* that it is needed to add */home/rustdev/.cargo/bin* to the PATH.

```dockerfile
# Install rustup
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs > rustup.sh
RUN sh rustup.sh -y && rm -f rustup.sh
ENV PATH /home/rustdev/.cargo/bin:$PATH
```

There is a bit more to say about the installation of *code-server*. Installing it requires you to be *sudoer* because the install script install a deb package. I use standalone method to install *code-server* because it install it in *~/.local* directory which allows installing extensions as *rustdev* user. After installation, The *~/.local* directory is owned by root user that is why the *chown* command change the ownership to be *rustdev*. To finish */home/rustdev/.local/bin* is added to the PATH as it contains the *code-server* executable. 

```dockerfile
# Install code-server 
RUN curl -fsSL https://code-server.dev/install.sh > install.sh
RUN sh install.sh --method=standalone
RUN rm -f install.sh
RUN sudo chown -R rustdev:rustdev /home/rustdev/.local
ENV PATH /home/rustdev/.local/bin:$PATH
```

Now that *code-server* is installed its time to install some extensions to make *code-server* user-friendly with Rust developers. I use *rustup* to install Rust toolchain components that are necessary for the extension I want. If you don't install these components now, this will be done each time you restart the container which is not the behavior I wanted. 

```dockerfile
# Install code-server extensions
## Install extensions dependencies
RUN rustup component add rust-analysis --toolchain stable-x86_64-unknown-linux-gnu
RUN rustup component add rust-src --toolchain stable-x86_64-unknown-linux-gnu
RUN rustup component add rls --toolchain stable-x86_64-unknown-linux-gnu
```

After reading a [post](https://dev.to/thiagomg/developing-in-rust-using-visual-studio-code-4kkl) by Thiago Massari Guedes I decided to trust him and to install [rust](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust), [rust-analyzer](https://marketplace.visualstudio.com/items?itemName=matklad.rust-analyzer) and [vscode-lldb](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb) extensions even if *rust-analyzer* is in alpha version and that the documentation of rust-analyzer says that it may cause conflicts with the [official Rust plugin](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust). We will see in use if Thiago's recommendations were good, I will make a feedback in the next posts as I progress in the discovery of Rust.

Rust extension is straightforward to install using *code-server* command line tool, *rust-analyzer* and *vscode-lldb* were more problematic.

```dockerfile
## Adds language support for Rust
RUN code-server --install-extension rust-lang.rust
```

My first attempt to install *rust-analyzer* was to simply run *code-server --install-extension matklad.rust-analyzer*. The problem is that it just installs the extension without downloading the *rust-analyzer* used by extension. As a consequence the executable is downloaded it each time the container is run when you try to use the extension. To prevent this behavior the *rust-analyzer* executable can be downloaded directly from github and put it where the extension expect to find it. 

Unfortunately, even taking these precautions, when you run the container and activate the extension by opening a first Rust project, the *rust-analyzer* extension ask for downloading the *rust-analyzer* binary. I have inspected the code of the *rust-analyzer* extension to understand the problem, if you are interested I created an [issue](https://github.com/rust-analyzer/rust-analyzer/issues/6428) in github.

A fix could be to modify the *package.json* of the extension by removing the version, but it is very dirty so I prefer keep the Dockerfile as these and wait the *rust-analyzer* team solves the problem as it is non blocking.

```dockerfile
## Alternative Rust Language Server
RUN code-server --install-extension matklad.rust-analyzer
## Download and Install rus-analyzer binary
RUN mkdir -p /home/rustdev/.local/share/code-server/User/globalStorage/matklad.rust-analyzer/
RUN curl -L https://github.com/rust-analyzer/rust-analyzer/releases/latest/download/rust-analyzer-linux -o /home/rustdev/.local/share/code-server/User/globalStorage/matklad.rust-analyzer/rust-analyzer-linux
RUN chmod u+x /home/rustdev/.local/share/code-server/User/globalStorage/matklad.rust-analyzer/rust-analyzer-linux
```

For *vscode-lldb* extension I have also tried to install it the same way I have installed *rust* and *rust-analyzer* extensions but I encounter problems trying to use it. After some research I find the *vscode-lldb* [issue 314](https://github.com/vadimcn/vscode-lldb/issues/314) so I tried the fix proposed. The fix consist in downloading manually from github the *vsix* extension file and install it from the command line, it works perfectly.

```dockerfile
## Debugging tools
## Install from the vsix instead of using extension id because of this issue: https://github.com/vadimcn/vscode-lldb/issues/314
RUN curl -L https://github.com/vadimcn/vscode-lldb/releases/download/v1.6.0/codelldb-x86_64-linux.vsix -o codelldb-x86_64-linux.vsix
RUN code-server --install-extension codelldb-x86_64-linux.vsix
RUN rm -f codelldb-x86_64-linux.vsix
```

After that I ended by installing the [better-toml](https://marketplace.visualstudio.com/items?itemName=bungcip.better-toml) extension useful to edit Cargo toml configuration files.

```dockerfile
## Toml extension, usefull to edit cargo configuration files
RUN code-server --install-extension bungcip.better-toml
```

Now everything is installed correctly it is time to remove the *sudo* package that is no more necessary. This can be done only as root user. Once *sudo* is removed the user can be set again to rustdev.

```dockerfile
# Remove sudo package
USER root
ENV SUDO_FORCE_REMOVE yes
RUN apt -y remove sudo
USER rustdev
```

To run *code-server* we need to overload the binding address and port is a way that the web UI can be access from the host and because this container is used for testing purpose we can also by-pass authentication mechanism.

```dockerfile
# Command to run code-server
CMD ["code-server", "--bind-addr=0.0.0.0:8080", "--auth=none"]	
```

To test if everything is working correctly simply run:

```bash
docker build -t rustcoder .
```

in the directory where the Dockerfile is saved.

## Run the docker image

This is how my working repository is organized:

```bash
├── docker-compose.yaml
├── docker-image
│   └── Dockerfile
├── projects
```

- The file *docker-compose.yaml* is a docker compose file that defines how to build and run our image. 
- The directory *docker-image* contains the Dockerfile.
- The directory projects is an empty repository.

I also need to precise that I use docker version 19.03.8 and docker-compose version 1.27.4.

Knowing that this is how my *docker-compose.yaml* looks like:

```yml
version: "3.8"
services:
  rustcoder:
    build: ./docker-image
    ports:
    - 8080:8080
    cap_add:
      - SYS_PTRACE
    security_opt:
      - seccomp:unconfined
    volumes:
      - ./projects:/home/rustdev/projects
```

The file format version is 3.8 which is a version compatible with my version of *docker* and *docker-compose*. 

```yaml
version: "3.8"
```

The file defines only one service, *rustcoder*. 

```yaml
services:
  rustcoder:
```

The image of the *rustcoder* service is built using the Dockerfile in the directory *./docker-image*. 

```yaml
    build: ./docker-image
```

The port 8080 of the container will be binded to the port 8080 of host. 

```yaml
    ports:
    - 8080:8080
```

The capability *SYS_PTRACE* is added because LLDB uses *ptrace* system calls.

 ```yaml
    cap_add:
      - SYS_PTRACE
 ```

To allow LLDB working correctly inside the container the *seccomp* security profile is set to *unconfined* which means that the container will be run without the default seccomp profile.

```yaml
    security_opt:
      - seccomp:unconfined
```

The host directory *./projects* is mount into the container */home/rustdev/projects* directory.

```yaml
    volumes:
      - ./projects:/home/rustdev/projects
```

Now running *rustcoder* service is simply running 

```bash
docker-compose up
```

in the directory where the *docker-compose.yaml* file is saved.

Once done you can access your brand new Rust development environment to *http://localhost:8080*

![](/home/sebastien/projects/OTHER/medium/learn-rust/01-learn-rust/imgs/code-server.png)

## Create a project with Cargo

To create a project with Cargo we need first to open a Terminal inside *code-server*

![](/home/sebastien/projects/OTHER/medium/learn-rust/01-learn-rust/imgs/open-terminal.png)

Once the Terminal is opened go to project directory and run

```bash
cargo new hello_world
```

Than click on *Open Folder* in the welcome screen or from the menu click *File > Open Folder* and then select */home/rustdev/projects/hello_world*. Due to the "bug" explained before the *rust-analyzer* extension should ask you for downloading the *rust-analyzer* binary.

Once done open the */home/rustdev/projects/hello_world/main.rs* file. Now you can run your first Rust application by simply clicking *Run* above the *main* function definition.

![](/home/sebastien/projects/OTHER/medium/learn-rust/01-learn-rust/imgs/main_rs.png)

## Debug your program with code-server

To illustrate how debugging work we will modify a little bit the *main* function by creating a *greetings* variable and putting a breakpoint. ![](/home/sebastien/projects/OTHER/medium/learn-rust/01-learn-rust/imgs/breakpoint.png)

To start the debugging session click on *Debug*, code-server automatically open the debug panel and the execution is stopped to the breakpoint. You can now click on the step over icon that appears in overlay.

![](/home/sebastien/projects/OTHER/medium/learn-rust/01-learn-rust/imgs/step_over.png)

## Conclusion

We have now a complete Rust development environment and we are ready to start learning Rust :). 