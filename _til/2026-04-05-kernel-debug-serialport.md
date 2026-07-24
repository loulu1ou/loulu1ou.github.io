---
layout: series
title: Kernel Debugging With Serial Port - Proxmox
order: 3
series_title: Today I Learned
---

This guide explains how to set up **Serial Kernel Debugging** between two Windows VMs on Proxmox. I use **VM 104** as the Debugger and **VM 112** as the Target which you can find in Proxmox.

### Phase 1: Proxmox Hardware Setup
You must add a virtual serial "wire" to both VMs.

1.  Log into the **Proxmox Web UI**.
2.  For **VM 104 (Debugger)**: 
    *   Go to **Hardware** -> **Add** -> **Serial Port**.
    *   Ensure **0** is selected and it says "socket". Click Add.
3.  For **VM 112 (Target)**:
    *   Go to **Hardware** -> **Add** -> **Serial Port**.
    *   Ensure **0** is selected and it says "socket". Click Add.
4.  **Start both VMs now.**

---

### Phase 2: Create the "Virtual Cable" (Proxmox Host)
Even though both VMs have serial ports, they aren't connected yet. We use the Proxmox Shell to bridge them.

1.  Open the **Shell** on your Proxmox Host (the physical server).
2.  Run this command to link the two virtual sockets:
    ```bash
    socat UNIX-CONNECT:/var/run/qemu-server/104.serial0 UNIX-CONNECT:/var/run/qemu-server/112.serial0 &
    ```
    *Note: This command must stay running in the background. If you reboot Proxmox, you must run this again.*

---

### Phase 3: Configure the TARGET VM (VM 112)
This tells Windows to send its internal kernel data out through the serial port.

1.  Log into **VM 112**.
2.  Open **Command Prompt (Admin)**.
3.  Run the following commands:
    ```cmd
    # Enable kernel debugging
    bcdedit /debug on

    # Configure the serial settings (COM1, 115200 baud)
    bcdedit /dbgsettings serial debugport:1 baudrate:115200
    ```
4.  **Restart VM 112.**

---

### Phase 4: Configure the DEBUGGER VM (VM 104)
This is where you will watch the logs and control the other VM.

1.  Log into **VM 104**.
2.  **Verify the Port:**
    *   Right-click the Start button -> **Device Manager**.
    *   Expand **Ports (COM & LPT)**.
    *   Confirm you see "Communications Port (**COM1**)". If it says COM2, remember that for the next step.
3.  Open **WinDbg Preview** (or WinDbg Classic).
4.  Go to **File** -> **Start Debugging** -> **Attach to Kernel**.
5.  Select the **COM** tab and enter:
    *   **Baud Rate:** `115200`
    *   **Port:** `COM1` (Match what you saw in Device Manager)
    *   Check the box for **Initial Break** (optional, but helpful).
6.  Click **OK**.
7.  The console should show: `Waiting to reconnect...`

---

### Phase 5: Testing the Connection
1.  While WinDbg is "Waiting" on VM 104, **Restart VM 112** (the Target).
2.  As VM 112 boots, you should see text scrolling in the WinDbg window on VM 104.
3.  To pause the target machine and start debugging, click the **Break** button (or press `Ctrl + Break`).
4.  The Target VM (112) should completely freeze. This is normal; the kernel is now under your control.
5.  Type `g` and press Enter in the WinDbg command line to let the Target VM continue running.
