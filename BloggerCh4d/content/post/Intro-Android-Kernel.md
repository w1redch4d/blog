+++
title = 'Intro Android Kernel'
date = 2025-04-11T13:52:58+05:30
draft = false
summary = 'A brief introduction on Zygote and ART'
+++
# Introduction to Android Kernel

## Overview of Android System Initialization and the Role of Zygote

After the kernel loads, Android begins its initialization process by parsing the init.rc file and starting the necessary native services. One of the early processes started is the execution of the app_process binary. This process eventually calls AndroidRuntime.callMain() with parameters that initialize the Zygote process, which in turn is critical for forking the System Server.

## 1. Startup via init.rc and app_process
- **Kernel and init.rc**:
    - Once the Android kernel is loaded, the system reads the `init.rc` file to execute startup instructions and launch native services.
    - Reference:
        - https://android.googlesource.com/platform/system/core/+/master/init/README.md

- **app_process Execution**:
    - The system then executes the `app_process` binary, which is located at:
        - https://android.googlesource.com/platform/frameworks/base/+/android13-qpr3-c-s1-release/cmds/app_process/
    - This binary sets up the runtime environment and later invokes the method `AndroidRuntime.callMain()`.

- **Invocation of AndroidRuntime.callMain()**:
    - AndroidRuntime.callMain() is called with two parameters:
        - `"com.android.internal.os.ZygoteInit"`
        - `"start-system-server"`
    The second parameter (`"start-system-server"`) signals that the Zygote process should initialize and later fork the System Server.

## 2. Processing Command-Line Arguments in ZygoteInit

- **Identification of the System Server Request**:
    - Inside the `main()` method of ZygoteInit, the system processes command-line arguments. When it finds the `"start-system-server"` parameter, it sets a flag `(startSystemServer)` to true.

- **Lazy Preload Consideration**:
    - Unless the `"--enable-lazy-preload"` flag is specified, the Zygote process preloads critical resources to reduce startup latency:
        - **Preloading Classes**:
            - Preloads commonly used Java classes to speed up application startup.
            - **Reference**:
                https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#282

        - **Preloading Resources**:
            - Loads system resources into memory.
            - **Reference**:
            https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/content/res/Resources.java;drc=9620962fc8f3aee01d42a0be7f9929bd77cc1f44;l=2738

        - **Preloading Graphics Drivers**:
            - Optionally initializes graphics drivers.
            - **Reference**:
                https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#219

        - **Preloading Shared Libraries**:
            - Loads shared libraries needed by various parts of the system.
            - **Reference**:
                https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#196

        - **Garbage Collection**:
            - After completing preloading, the Zygote process calls [gcAndFinalize()](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#460) to run garbage collection and free up any unused memory.

## 3. Starting the Zygote Server

- **Initialization:**
    - The Zygote server ([zygoteServer](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#797)) is initialized via [`new ZygoteServer(isPrimaryZygote)`](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#881).

## 4. Forking the System Server Process

If `startSystemServer` is true, the forkSystemServer method is [invoked](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#884):

- Capabilities Setup: Calculates the capabilities required for the System Server (e.g., networking, system time management).
- Command Construction: Constructs a command-line argument array for the System Server.
- Forking: The Zygote forks a new process for the System Server using [Zygote.forkSystemServer()](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#672).

- If the [forked process `(pid == 0)`](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#771):

    - The Zygote server socket is closed.
    - [handleSystemServerProcess](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#466) is executed:
        - Sets the UID, GID, and process group for the System Server.
        - Prepares the environment (e.g., classpath, permissions).
        - Passes control to the `SystemServer` class by invoking [zygoteInit](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#536).

- In the parent process (`Zygote`):
    - The Zygote server enters a command socket loop ([zygoteServer.runSelectLoop(abiList)](https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/os/ZygoteInit.java#898)), waiting for commands to spawn additional child processes.

Here is a graph diagram of Zygote's execution flow:
{{< mermaid >}}
graph TD
    A[ZygoteInit.main] --> B[Parse Command-line Arguments]
    B --> C[Check 'start-system-server' Parameter]
    
    C -->|No Lazy Preload| D[Preload Resources and Classes]
    
    D --> E[Initialize ZygoteServer]

    C -->|Lazy Preload| E

    E -->| 'start-system-server' is True | F[Fork System Server]
    E -->| 'start-system-server' is False | G[Parent Process<br>Run Command Socket Loop]

    F -->|Child Process pid == 0| H[Child Process<br>Handle System Server]
    H --> I[Setup Environment and Invoke SystemServer]

    F -->|Parent Process pid != 0| G
{{< /mermaid >}}
