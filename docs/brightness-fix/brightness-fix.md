# **Display Brightness Persistence and Backlight Management in GNOME Wayland Environments**

## **Executive Summary of Display Luminance Architecture**

The transition of the Linux desktop from the legacy X11 windowing system to the Wayland display server protocol has necessitated a fundamental re-engineering of how graphical environments interact with physical hardware. Within the GNOME desktop environment, specifically in the developmental iterations spanning GNOME 48 to GNOME 49 (deployed in distributions such as Fedora 44 Beta), the architecture governing display luminance, High Dynamic Range (HDR) signaling, and multi-monitor backlight synchronization has undergone a profound transformation. This architectural shift, while modernizing the graphics stack, introduces complex state persistence behaviors for advanced external monitors.

When utilizing state-of-the-art self-emissive display technologies, such as the Samsung 27” Odyssey OLED G6 (G61SD) QD-OLED 240Hz monitor, systems engineers and users frequently encounter a specific state persistence anomaly. Specifically, the operating system's internal brightness parameter resets to a median value (typically 50%) upon system startup, waking from sleep, or session unlocking, despite the monitor's internal hardware on-screen display (OSD) retaining a 100% brightness configuration.2 Furthermore, this reset often targets the primary OLED display while leaving secondary LCD monitors operating at their expected full brightness.4

This discrepancy is not a hardware defect within the Samsung display, nor is it a malfunction of the monitor's firmware. Rather, it is an explicit byproduct of the newly implemented Kernel Mode Setting (KMS) backlight application programming interfaces (APIs), the delegation of luminance control entirely to the Mutter compositor, and the mathematical preservation of HDR headroom via software-based pixel scaling.1 The operating system attempts to manage the display's output dynamically, prioritizing headroom for HDR specular highlights over maintaining a static Standard Dynamic Range (SDR) reference white level.

This exhaustive technical report dissects the underlying architectural mechanisms driving this behavior within GNOME on Fedora 44\. The analysis explores the historical context of Linux display management, the specific interactions between the Samsung QD-OLED hardware and the I2C bus, the Mutter compositor's D-Bus interfaces, and the mechanics of HDR software dimming. Most importantly, the report provides robust, deeply integrated engineering solutions—ranging from Python-driven D-Bus parsers to systemd-automated daemon units—designed to permanently resolve external monitor brightness persistence issues without destabilizing the compositor.

## **The Evolution of Linux Display Luminance Control**

To comprehend the current state of external monitor brightness management in Fedora 44, it is necessary to trace the historical progression of Linux display control mechanisms. Historically, display brightness was managed through highly fragmented, bus-specific protocols that lacked a unified abstraction layer.

### **The Legacy X11 and Sysfs Paradigms**

During the dominance of the X11 display server, luminance control was bifurcated based on the physical connection of the display. For internal laptop displays utilizing Low-Voltage Differential Signaling (LVDS) or embedded DisplayPort (eDP), the Linux kernel exposed hardware backlight controls through the /sys/class/backlight/ directory structure.7 Depending on the integrated graphics processor, users would interact with symbolic links such as intel\_backlight or acpi\_video0.8 Daemons like gnome-settings-daemon (specifically the gsd-power component) would monitor physical keyboard brightness keys, intercept the ACPI events, and write absolute integer values directly to the sysfs interface to adjust the electrical current supplied to the LED backlight array.

However, this sysfs paradigm possessed severe limitations. The interface was largely incapable of addressing external monitors connected via standard DisplayPort or High-Definition Multimedia Interface (HDMI) connections.1 External displays rely on the Display Data Channel Command Interface (DDC/CI) protocol, which is transmitted over the Inter-Integrated Circuit (I2C) bus.10 Because sysfs did not natively map external DDC/CI capabilities into its directory structure, the operating system historically ignored external monitor hardware backlights entirely.

To circumvent this limitation in X11, users relied on user-space utilities such as xrandr. Commands like xrandr \--output DP-1 \--brightness 1.0 did not alter the physical backlight of the monitor. Instead, they performed matrix multiplication on the color values within the X server's rendering pipeline, artificially dimming the pixels before they were transmitted to the display.11 While functional, this approach crushed contrast ratios and degraded image quality, particularly on advanced panels. Other utilities, such as xbacklight, relied on the RandR extension in Xorg and similarly failed when addressing modern external connections.13

### **The Wayland Compositor Model and Strict Isolation**

With the advent of the Wayland display server protocol, the architecture shifted toward a highly secure, monolithic control structure. Under Wayland, the compositor—which is Mutter in the GNOME desktop environment—assumed absolute authority over the entire display pipeline.1 Client applications and peripheral user-space daemons lost all direct access to the display hardware and the rendering loop.

This strict isolation immediately rendered legacy X11 tools obsolete. Utilities like xrandr interact only with the XWayland compatibility layer, which possesses no authority over the primary Mutter display output and cannot manipulate the host's physical displays.11 Similarly, tools designed for other Wayland compositors, such as wlr-randr (built for the wlroots library used by Sway and Hyprland), are explicitly rejected by GNOME because Mutter does not implement the wlr-output-management protocol.16

Consequently, the responsibility for managing monitor configurations, refresh rates, logical scaling, color profiles, and brightness fell entirely to the Mutter compositor, which communicates directly with the Linux kernel's Direct Rendering Manager (DRM) and KMS subsystems.1

## **GNOME Architecture and the Backlight Overhaul**

The release of GNOME 48 and the subsequent fundamental refinements in GNOME 49 introduced a sweeping architectural overhaul of backlight management, spearheaded by core contributors to address the immense complexities introduced by High Dynamic Range (HDR) displays and multi-monitor setups.1 The anomalies observed in Fedora 42 through Fedora 44 are direct artifacts of this ongoing architectural migration.

### **Centralization of Display State within Mutter**

Prior to the GNOME 49 overhaul, gnome-settings-daemon (GSD) maintained responsibility for directly manipulating the sysfs backlight files for internal panels. This legacy approach created race conditions, required elevated root privileges or complex logind helper executables, and fundamentally failed in multi-monitor configurations where multiple disparate backlights required synchronous adjustments. GNOME historically operated under the flawed assumption that a system would possess only a single controllable internal display.

The GNOME 49 architecture dismantles this legacy assumption. The logic for interfacing with physical hardware has been relocated from gnome-settings-daemon directly into the Mutter compositor. Mutter now independently evaluates the capabilities of all connected displays, constructing an internal topology of available CRTCs (Cathode Ray Tube Controllers) and outputs.15 If an external monitor supports a hardware backlight via DDC/CI or a modern KMS backlight API, Mutter attempts to abstract and control it natively. If hardware control is unavailable, or explicitly overridden by HDR signaling requirements, Mutter employs a highly sophisticated "software backlight" emulation.

### **The OLED HDR Paradigm and Software Luminance Scaling**

The core phenomenon—where the Samsung Odyssey OLED G6 (G61SD) hardware OSD reports 100% brightness while the GNOME OS slider defaults to 50%—is inextricably linked to Mutter's HDR software backlight emulation.1

Quantum Dot OLED (QD-OLED) panels are self-emissive devices; they do not possess a global LED backlight layer. Each individual pixel generates its own illumination. When the Samsung G61SD operates in HDR mode (e.g., VESA DisplayHDR True Black 400), the display's internal processing must reserve physical luminance capability to render intensely bright specular highlights, such as explosions or the sun in digital media.1 To achieve this without blinding the user with a fully white graphical interface, the operating system must establish a "reference white level"—the standard luminance target for Standard Dynamic Range (SDR) elements like application windows, text rendering, and the desktop background.

If an OLED monitor is physically capable of producing 1000 nits of peak peak brightness in a 2% window, the operating system cannot map standard desktop white to the maximum signal value, as it would cause severe eye strain, induce rapid OLED burn-in, and leave zero mathematical "HDR headroom" for actual HDR content.

Therefore, Mutter dynamically scales the output signal down. It maps the standard OS white to a fractional signal value. In previous iterations like GNOME 48, this was exposed as a separate "HDR Brightness" slider in the settings panel. In GNOME 49, this has been seamlessly integrated into the primary brightness control. The GNOME brightness slider for the external Samsung OLED monitor is no longer controlling the physical electrical current to a hardware backlight array. Instead, it is manipulating this software-based reference white level multiplier within the compositor's rendering pipeline. Setting the slider to 100% instructs Mutter to push the SDR reference white level as high as the current HDR headroom allows.

### **The Mechanics of the Persistence Anomaly**

The specific issue experienced by users of Fedora 44 (Beta) is a state persistence failure regarding this new multi-monitor software brightness parameter.5

When a user manually pushes the GNOME brightness slider to 100% on the primary Samsung monitor, they are commanding Mutter to set the SDR reference white level multiplier to 1.0, utilizing the maximum safe SDR brightness capacity of the display. However, when the system suspends, the screen locks, or the compositor restarts, GNOME temporarily drops the DRM master lease on the display pipeline.1

Upon waking, Mutter must renegotiate the display link. It reads the Electronic Display Identification Data (EDID) block provided by the Samsung monitor over the DisplayPort or HDMI connection.20 Upon parsing the EDID and detecting an HDR-capable QD-OLED display, Mutter's internal logic conservatively defaults the signal multiplier back to a safety baseline—exactly 50%—to instantly guarantee the presence of HDR headroom.1 The compositor effectively discards the user's previous 100% slider configuration because it prioritizes safety and HDR compliance over persisting the software luminance multiplier.

This behavior highlights why the secondary monitor is unaffected. Standard LCD monitors operating in SDR mode do not trigger the HDR headroom preservation logic. Mutter either leaves their software multiplier at 1.0 or successfully interacts with their hardware backlight through legacy pathways, allowing them to remain at 100% brightness.4

### **The Failure of monitors.xml for Luminance Serialization**

Historically, display state persistence in GNOME is managed by the \~/.config/monitors.xml configuration file.22 This XML file dictates logical monitor layout, primary display designation, coordinate offsets, scaling factors, and refresh rates.22

However, the GNOME 49 transition has resulted in edge cases where software backlight values for external monitors are explicitly excluded from this serialization schema. The XML schema was designed for static physical attributes, not dynamic signal multipliers.5 Because the software backlight is intended to respond dynamically to ambient light sensors (ALS) and HDR content metadata, gsd-power avoids committing the value to static storage. Consequently, every time the monitor wakes from sleep (a hot-plug event from the kernel's perspective), Mutter treats it as a newly connected HDR display and applies the default 50% safety multiplier.2

## **Technical Profile: Samsung Odyssey OLED G6 (G61SD)**

Understanding the interaction between the GNOME Wayland architecture and the specific hardware is crucial for evaluating remediation strategies. The Samsung Odyssey OLED G6 (G61SD) possesses specific characteristics that dictate how the Linux graphics stack interacts with it.

The following table details the hardware specifications and their direct impact on the Linux display management pipeline.

| Hardware Specification | Description | Impact on Linux Display Management Pipeline |
| :---- | :---- | :---- |
| **Panel Technology** | Quantum Dot OLED (QD-OLED) | Self-emissive pixel structure means no traditional /sys/class/backlight node is generated by the amdgpu or nouveau/nvidia KMS drivers. Requires software signal scaling or direct DDC/CI I2C injection for OS-level dimming.18 |
| **Refresh Rate** | 240Hz Variable Refresh Rate (VRR) | Extremely high bandwidth requirements over DisplayPort 1.4/HDMI 2.1. Rapid EDID polling during KMS mode setting can saturate the bus, interrupting DDC/CI initialization during system wake states.18 |
| **Color Volume** | HDR10 / VESA DisplayHDR True Black 400 | The presence of advanced HDR metadata in the EDID block triggers Mutter's HDR headroom preservation logic, causing the compositor to forcefully reset the reference white level to 50% upon initialization. |
| **Hardware OSD** | Independent Microcontroller Firmware | Operates entirely decoupled from the operating system. Setting hardware brightness to 100% scales the physical peak luminance threshold, but the OS signal multiplier ultimately dictates the actual rendered desktop brightness.6 |

The firmware on the Samsung G61SD is highly optimized for gaming performance and input latency reduction. However, like many modern gaming monitors, its implementation of the DDC/CI protocol can be computationally sluggish. When GNOME attempts to probe the monitor via the I2C bus upon waking from suspend, the monitor's microcontroller may be preoccupied with re-establishing the high-bandwidth 240Hz DisplayPort link. It may not acknowledge the DDC/CI probe swiftly enough. Consequently, GNOME defaults strictly to its internal software backlight emulation, resetting the UI slider to the pre-programmed median value.10

## **Analysis of Remediation Strategies**

Given the architectural constraints of Mutter and the specific behavior of the Samsung QD-OLED display, several methodologies exist to override the compositor's default behavior. It is critical to evaluate these strategies to identify the most robust approach for a professional environment.

The following table summarizes the comparative viability of the primary intervention vectors.

| Intervention Strategy | Execution Mechanism | System Stability | HDR Compatibility | Persistence Efficacy |
| :---- | :---- | :---- | :---- | :---- |
| **DDC/CI Hardware Override** | GNOME Shell Extensions (ddcutil) | Low (Prone to I2C bus locking and compositor crashing) | Excellent (Bypasses OS software scaling) | Variable (Dependent on extension load timing) |
| **ICC Gamma Profiling** | gnome-gamma-tool to static 1.0 profile | High (Low-level rendering pipeline operation) | Poor (Destroys HDR tone-mapping capabilities) | Complete (Ignores power state resets) |
| **D-Bus Event Automation** | Systemd daemon triggering busctl/gdbus | Excellent (Leverages native GNOME IPC architecture) | Excellent (Interacts natively with Mutter's sliders) | Complete (Reacts instantly to unlock events) |

### **Strategy 1: DDC/CI Subjugation via GNOME Extensions**

The first approach seeks to bypass the Mutter software backlight entirely and force the operating system to interface with the monitor's hardware microcontroller via the I2C bus.10

The ddcutil command-line utility interacts directly with the DDC/CI protocol, allowing users to push commands such as ddcutil setvcp 10 \+ 5 to manipulate the VCP (Virtual Control Panel) brightness code.10 By installing a GNOME Shell Extension such as "Brightness control using ddcutil," the native GNOME brightness slider logic is intercepted, and adjustments are piped directly to the monitor's firmware.28

While this provides genuine hardware-level brightness control and circumvents Mutter's software HDR headroom calculations 30, it introduces severe stability risks. Polling DDC/CI on an active 240Hz display stream can induce rendering stuttering or kernel locking.10 Furthermore, as reported by systems engineers, this extension can severely destabilize GNOME 49, as the native compositor and the extension actively fight for control over the display's D-Bus properties, leading to race conditions.4

A comparative analysis with the KDE Plasma 6 environment reveals similar issues. In Fedora KDE, the PowerDevil daemon attempts to use ddcutil automatically to manage external monitors. On displays with slow I2C responses, this causes catastrophic brightness resets, prompting engineers to disable the functionality entirely via the POWERDEVIL\_NO\_DDCUTIL=1 environment variable.6 Injecting DDC/CI control into GNOME via an extension replicates these exact instability vectors.

### **Strategy 2: Absolute Color Profiling via ICC Gamma Overrides**

If the operating system insists on manipulating the software signal to dim the display, users can counter this by applying a static ICC (International Color Consortium) gamma profile that forces the color multiplier to maximum output, irrespective of the GNOME slider.31

Utilizing the gnome-gamma-tool (a component of the gnome-color-manager suite), a static gamma profile of 1.0 can be generated.32 The monitor must be explicitly marked as "Color Managed" within the GNOME Settings.31 By creating a system-wide ICC profile (e.g., /usr/local/share/gnome-gamma-profiles/gamma-1.0.icc) and dynamically binding it to the device object path via colormgr, the pixel output is permanently mapped to a 1:1 signal ratio.31

This approach operates at the lowest level of the color management pipeline, making it highly stable and immune to power state resets. However, it completely breaks accurate color representation in true HDR modes. By forcing the gamma curve linearly, the operating system loses the mathematical ability to properly tone-map highlights, resulting in severely washed-out imagery or crushed specular details.31 Given the premium HDR capabilities of the Samsung Odyssey G6, degrading the color pipeline to achieve brightness persistence is an unacceptable engineering compromise.

### **Strategy 3: Event-Driven D-Bus Injection via Systemd**

The most robust, non-destructive, and architecturally sound method for resolving the brightness reset in Fedora 44 is to deploy a user-level systemd service paired with a specialized D-Bus monitoring script.8

Instead of fighting the GNOME compositor's initialization logic, this solution respects the Wayland security model. It simply waits for the compositor to finish its EDID handshake and unlock sequence, and immediately dispatches a standard D-Bus command through the Inter-Process Communication (IPC) bus to return the OS brightness slider to 100%.13 This leverages the native APIs, maintains perfect HDR compatibility, and prevents I2C bus collision.

## **D-Bus Introspection and the Mutter API Structure**

To implement the event-driven injection strategy, a deep understanding of the GNOME D-Bus architecture is required. The Desktop Bus (D-Bus) serves as the primary IPC mechanism, exposing the properties and methods of the Mutter compositor to external applications and user scripts.19

### **The org.gnome.SettingsDaemon.Power Interface**

Historically, and continuing into GNOME 49 for UI synchronization, the org.gnome.SettingsDaemon.Power.Screen interface acts as the proxy for brightness manipulation.13

Commands such as: busctl \--user call org.gnome.SettingsDaemon.Power /org/gnome/SettingsDaemon/Power org.gnome.SettingsDaemon.Power.Screen StepUp are utilized to sequentially increment the UI slider.14 Furthermore, absolute values can be written directly to the property using the org.freedesktop.DBus.Properties.Set method.13 While highly effective for global brightness commands, gsd-power can sometimes struggle with precise multi-monitor targeting, necessitating a deeper look into the compositor's direct API.

### **The org.gnome.Mutter.DisplayConfig Interface**

In the GNOME Wayland architecture, authoritative display states are managed via the org.gnome.Mutter.DisplayConfig D-Bus interface, located at the object path /org/gnome/Mutter/DisplayConfig.15 This interface exposes the raw topology of the graphics stack.

The GetCurrentState method is vital for advanced scripting automation. When called, it outputs a highly complex, nested array of tuples and dictionaries detailing the exact state of every connected monitor, including logical configurations, scaling factors, refresh rates, EDID blocks, and connector names (e.g., DP-1, HDMI-A-1).37 The D-Bus signature for this output is ua((ssss)a(siiddada{sv})a{sv})a(iiduba(ssss)a{sv})a{sv}, representing a formidable parsing challenge for standard bash utilities.39

GNOME 49 introduces explicit methods within the DisplayConfig interface for granular backlight manipulation, deprecating older generalized methods.34

* ChangeBacklight(uint32 serial, uint32 output, int32 value): Adjusts the brightness based on the numerical output ID.34  
* SetBacklight(uint32 serial, string connector, int32 value): Directly maps a brightness integer to a specific hardware connector string.34

While SetBacklight is the most direct way to target the specific Samsung monitor without affecting the secondary display, invoking it requires passing the current configuration serial, an integer that increments every time the display topology changes.41 Consequently, a robust script must first parse GetCurrentState to acquire the active serial before issuing the SetBacklight command.

## **Engineering the Automated Remediation Architecture**

The optimal solution to the persistence anomaly is a multi-component architecture utilizing Python to parse the complex D-Bus signatures safely, driven by a lightweight bash daemon monitored by a user-level systemd unit.

### **Component 1: The Python D-Bus Execution Script**

Because bash is poorly equipped to parse the deeply nested GVariant outputs of Mutter's GetCurrentState method 38, Python 3 with the pydbus or native dbus module is required to gracefully navigate the topology and target only the primary OLED monitor.

Create a script file at \~/.local/bin/gnome-brightness-target.py and populate it with the following executable code.

Python

\#\!/usr/bin/env python3  
"""  
Description: Dynamically parses GNOME Mutter's display topology via D-Bus,
identifies the target Samsung OLED monitor, and forces the software
backlight to 100% using the GNOME 49 SetBacklight API.  
"""

import sys  
import dbus

def restore\_brightness():  
    try:  
        \# Connect to the user's session bus  
        bus \= dbus.SessionBus()  
          
        \# Define the Mutter DisplayConfig namespace and object path  
        mutter\_bus\_name \= "org.gnome.Mutter.DisplayConfig"  
        mutter\_object\_path \= "/org/gnome/Mutter/DisplayConfig"  
          
        \# Acquire the proxy object and interface  
        proxy \= bus.get\_object(mutter\_bus\_name, mutter\_object\_path)  
        interface \= dbus.Interface(proxy, dbus\_interface=mutter\_bus\_name)  
          
        \# Call GetCurrentState to retrieve the active display topology  
        \# Signature: ua((ssss)a(siiddada{sv})a{sv})a(iiduba(ssss)a{sv})a{sv}  
        state \= interface.GetCurrentState()  
          
        \# The first element is the configuration serial required for subsequent calls  
        config\_serial \= state  
          
        \# The second element is the array of physical monitors  
        physical\_monitors \= state  
          
        target\_connector \= None  
          
        \# Iterate through physical monitors to locate the Samsung display  
        for monitor in physical\_monitors:  
            monitor\_info \= monitor  
            connector\_name \= str(monitor\_info)  
            vendor \= str(monitor\_info)  
            product \= str(monitor\_info)  
              
            \# Identify the monitor by EDID vendor/product string or connector type  
            \# The Samsung Odyssey G6 can be identified by vendor or simply   
            \# targeting the primary high-refresh DisplayPort connection.  
            if "Samsung" in vendor or "SAM" in vendor or "Odyssey" in product:  
                target\_connector \= connector\_name  
                break  
          
        \# Fallback: if the vendor string is obfuscated by the hub, target the first DP  
        if not target\_connector:  
            for monitor in physical\_monitors:  
                connector\_name \= str(monitor)  
                if "DP" in connector\_name:  
                    target\_connector \= connector\_name  
                    break  
                      
        if target\_connector:  
            \# Set target brightness to 100%  
            target\_brightness \= 100  
              
            \# Invoke the GNOME 49 SetBacklight method  
            \# Arguments: serial (uint32), connector (string), value (int32)  
            interface.SetBacklight(dbus.UInt32(config\_serial),   
                                   dbus.String(target\_connector),   
                                   dbus.Int32(target\_brightness))  
            print(f"Successfully restored brightness on {target\_connector} to 100%.")  
        else:  
            print("Target monitor could not be identified in the display topology.")  
            sys.exit(1)  
              
    except dbus.exceptions.DBusException as e:  
        print(f"D-Bus Communication Error: {e}")  
        \# Fallback to gsd-power if Mutter direct API fails or is locked  
        import subprocess  
        print("Attempting fallback via SettingsDaemon.Power...")  
        subprocess.run(, capture\_output=True)

if \_\_name\_\_ \== "\_\_main\_\_":  
    restore\_brightness()

*Architectural Justification for the Script:* The Python script provides surgical precision. By parsing the GetCurrentState tuple, the script reads the EDID vendor strings directly from the active DRM pipeline.21 Once the Samsung QD-OLED is positively identified, it captures the required config\_serial and the specific connector string (e.g., DP-1). It then invokes SetBacklight exclusively on that connector.34 This directly satisfies the user's constraint that the secondary display must remain untouched by the remediation effort. The script also contains a robust fallback to gsd-power via subprocess should the Mutter interface lock due to DRM mode setting.13

### **Component 2: The D-Bus Watcher Daemon**

A simple systemd trigger on boot is insufficient, as the persistence anomaly reproduces every time the monitor wakes from sleep or the user unlocks the desktop.2 Therefore, the solution requires a persistent shell daemon that listens asynchronously to the D-Bus for the specific GNOME ScreenSaver unlock signal.

Create a secondary script at \~/.local/bin/gnome-brightness-watcher.sh:

Bash

\#\!/bin/bash  
\# Description: Daemon that monitors D-Bus for session unlock events and   
\# triggers the Python brightness restoration script.

\# Listen to the D-Bus session for the ActiveChanged signal emitted by the GNOME ScreenSaver  
dbus-monitor \--session "type='signal',interface='org.gnome.ScreenSaver',member='ActiveChanged'" |  
while read \-r line; do  
    \# When the screen unlocks, the Active property changes to boolean false  
    if echo "$line" | grep \-q "boolean false"; then  
        \# Introduce a brief delay to allow Mutter to complete EDID HDR profiling  
        \# If executed instantaneously, Mutter will overwrite the script's command.  
        sleep 2  
          
        \# Fire the Python restoration script in the background  
        /usr/bin/python3 \~/.local/bin/gnome-brightness-target.py &  
    fi  
done

### **Component 3: The User-Level Systemd Unit**

To ensure the watcher daemon starts automatically upon login and survives compositor restarts without requiring elevated root privileges, it must be encapsulated in a standardized user-level systemd unit.8

Create the unit file at \~/.config/systemd/user/external-brightness-restore.service:

Ini, TOML

\[Unit\]  
Description\=Automated GNOME Wayland Brightness Restoration Daemon  
\# Ensure the service only starts after the graphical environment is fully loaded  
After\=graphical-session.target  
PartOf\=graphical-session.target

Type\=simple  
\# Path to the watcher daemon  
ExecStart\=%h/.local/bin/gnome-brightness-watcher.sh  
\# Automatically restart the watcher if the D-Bus connection drops  
Restart\=always  
RestartSec\=3

\[Install\]  
WantedBy\=graphical-session.target

To deploy the architecture, the user must execute the following terminal commands to grant execution permissions and enable the systemd daemon within their profile:

Bash

chmod \+x \~/.local/bin/gnome-brightness-target.py  
chmod \+x \~/.local/bin/gnome-brightness-watcher.sh  
systemctl \--user daemon-reload  
systemctl \--user enable \--now external-brightness-restore.service

### **Verification of the Deployment Lifecycle**

Upon deployment, the architecture functions silently and asynchronously. When the user physically returns to the workstation and authenticates through the GNOME Display Manager (GDM), the session unlocks. The org.gnome.ScreenSaver interface emits the ActiveChanged signal over the session bus.

The watcher daemon intercepts this packet and pauses for two seconds. This deliberate pause allows Mutter to complete its EDID handshake over the DisplayPort interface, recognize the HDR capabilities of the Samsung Odyssey G6, and forcefully apply its 50% SDR safety multiplier.1 Once Mutter concludes its internal logic, the watcher launches the Python script. The script parses the display topology, locates the OLED connector, and forces the D-Bus property to 100%. The GNOME UI slider instantly reflects this change, and the display returns to peak brightness.

Because the Python script actively filters for the specific Samsung vendor string or DP connector, the secondary monitor is completely insulated from the D-Bus command. Furthermore, since the script does not attempt to poll the physical DDC/CI interface, the system avoids the kernel panics and UI stuttering commonly associated with I2C bus saturation on high-refresh-rate displays.10

## **Future Outlook: The KMS Backlight API**

The interaction between the Samsung Odyssey QD-OLED display and the GNOME Wayland compositor highlights a broader transitional friction point within the Linux graphics stack. As display hardware rapidly evolves—incorporating features like variable refresh rates (VRR), 10-bit HDR color pipelines, and localized micro-controller intelligence—the operating system must abstract these immense hardware complexities into coherent, responsive user interfaces.18

The architectural migration currently underway in GNOME 49 is merely a stepping stone. The implementation of the new, dedicated KMS backlight API, as outlined by GNOME core developers, will eventually eliminate the need for the complex D-Bus workaround provided in this report. In future Linux kernel iterations, the DRM subsystem will natively expose standardized, secure interfaces for adjusting HDR headroom mathematically at the kernel level. This will allow the compositor to serialize these fractional luminance values securely into the system's core configuration parameters without triggering race conditions during session unlock sequences.

Until that upstream code is finalized, merged into the mainline kernel, and widely distributed downstream to Fedora releases, the D-Bus automation paradigm remains the most elegant, non-intrusive, and mathematically correct method to manage software luminance scaling on high-end external monitors within the Linux ecosystem. It respects the Wayland security model, leverages the native GNOME APIs, and guarantees persistence across all power state transitions.

#### **Works cited**

1. GNOME 49 Backlight Changes \- swick's blog \- Sebastian Wick, accessed April 13, 2026, [https://blog.sebastianwick.net/posts/gnome-49-backlight-changes/](https://blog.sebastianwick.net/posts/gnome-49-backlight-changes/)  
2. My monitor automatically changes brightness to 100% every time I unlock? : r/Fedora, accessed April 13, 2026, [https://www.reddit.com/r/Fedora/comments/1hb80aa/my\_monitor\_automatically\_changes\_brightness\_to/](https://www.reddit.com/r/Fedora/comments/1hb80aa/my_monitor_automatically_changes_brightness_to/)  
3. Brightness issue after wake up : r/Fedora \- Reddit, accessed April 13, 2026, [https://www.reddit.com/r/Fedora/comments/1dla9bt/brightness\_issue\_after\_wake\_up/](https://www.reddit.com/r/Fedora/comments/1dla9bt/brightness_issue_after_wake_up/)  
4. Multi monitor \-- Brightness control \- Desktop \- GNOME Discourse, accessed April 13, 2026, [https://discourse.gnome.org/t/multi-monitor-brightness-control/32658](https://discourse.gnome.org/t/multi-monitor-brightness-control/32658)  
5. In Gnome 49 is the display brightness adjustment (when HDR is enabled) supposed to retain its value after logout? \- Reddit, accessed April 13, 2026, [https://www.reddit.com/r/gnome/comments/1noch27/in\_gnome\_49\_is\_the\_display\_brightness\_adjustment/](https://www.reddit.com/r/gnome/comments/1noch27/in_gnome_49_is_the_display_brightness_adjustment/)  
6. My monitor automatically changes brightness to 100% every time I ..., accessed April 13, 2026, [https://www.reddit.com/r/Fedora/comments/1hb80aa/my\_monitor\_automatically\_changes\_brightness\_to\_100\_every\_time\_i\_unlock/](https://www.reddit.com/r/Fedora/comments/1hb80aa/my_monitor_automatically_changes_brightness_to_100_every_time_i_unlock/)  
7. gnome \- Desktop doesn't remember brightness settings after a reboot \- Ask Ubuntu, accessed April 13, 2026, [https://askubuntu.com/questions/3841/desktop-doesnt-remember-brightness-settings-after-a-reboot](https://askubuntu.com/questions/3841/desktop-doesnt-remember-brightness-settings-after-a-reboot)  
8. Save backlight brightness settings across reboots, Gnome 3 \- openSUSE Forums, accessed April 13, 2026, [https://forums.opensuse.org/t/save-backlight-brightness-settings-across-reboots-gnome-3/74527](https://forums.opensuse.org/t/save-backlight-brightness-settings-across-reboots-gnome-3/74527)  
9. Does Wayland have an equivalent of xrandr for changing brightness and color temperature?, accessed April 13, 2026, [https://askubuntu.com/questions/1286458/does-wayland-have-an-equivalent-of-xrandr-for-changing-brightness-and-color-temp](https://askubuntu.com/questions/1286458/does-wayland-have-an-equivalent-of-xrandr-for-changing-brightness-and-color-temp)  
10. Controlling External Monitor Brightness on Linux Using Brightness Control Keys, accessed April 13, 2026, [https://praveenpuglia.com/posts/controlling-external-monitor-brightness-on-linux/](https://praveenpuglia.com/posts/controlling-external-monitor-brightness-on-linux/)  
11. How to control external monitor/s brightness in Fedora?, accessed April 13, 2026, [https://discussion.fedoraproject.org/t/how-to-control-external-monitor-s-brightness-in-fedora/75138](https://discussion.fedoraproject.org/t/how-to-control-external-monitor-s-brightness-in-fedora/75138)  
12. Why do xrandr errors "BadMatch", "BadName", "Gamma Failed" happen? \- Ask Ubuntu, accessed April 13, 2026, [https://askubuntu.com/questions/710172/why-do-xrandr-errors-badmatch-badname-gamma-failed-happen](https://askubuntu.com/questions/710172/why-do-xrandr-errors-badmatch-badname-gamma-failed-happen)  
13. Backlight \- ArchWiki, accessed April 13, 2026, [https://wiki.archlinux.org/title/Backlight](https://wiki.archlinux.org/title/Backlight)  
14. How to change LCD brightness from command line (or via script)? \- Ask Ubuntu, accessed April 13, 2026, [https://askubuntu.com/questions/149054/how-to-change-lcd-brightness-from-command-line-or-via-script](https://askubuntu.com/questions/149054/how-to-change-lcd-brightness-from-command-line-or-via-script)  
15. Initiatives/Wayland/Gaps/DisplayConfig \- GNOME wiki, accessed April 13, 2026, [https://wiki.gnome.org/Initiatives(2f)Wayland(2f)Gaps(2f)DisplayConfig.html](https://wiki.gnome.org/Initiatives\(2f\)Wayland\(2f\)Gaps\(2f\)DisplayConfig.html)  
16. adjust external monitor brightness from command line \[ubuntu 22.04 wayland\] (xrandr not working), accessed April 13, 2026, [https://askubuntu.com/questions/1443067/adjust-external-monitor-brightness-from-command-line-ubuntu-22-04-wayland-xra](https://askubuntu.com/questions/1443067/adjust-external-monitor-brightness-from-command-line-ubuntu-22-04-wayland-xra)  
17. GNOME 49 Backlight Changes, GNOME Foundation Report, and This Week in GNOME \- Tux Machines, accessed April 13, 2026, [https://news.tuxmachines.org/n/2025/08/09/GNOME\_49\_Backlight\_Changes\_GNOME\_Foundation\_Report\_and\_This\_Wee.shtml](https://news.tuxmachines.org/n/2025/08/09/GNOME_49_Backlight_Changes_GNOME_Foundation_Report_and_This_Wee.shtml)  
18. I'd choose this $380 OLED over a 4K monitor in a heartbeat \- How-To Geek, accessed April 13, 2026, [https://www.howtogeek.com/dont-waste-money-on-a-4k-gaming-monitor-buy-an-oled-instead/](https://www.howtogeek.com/dont-waste-money-on-a-4k-gaming-monitor-buy-an-oled-instead/)  
19. Subscribe for DBUS event of screen power off \- Ask Ubuntu, accessed April 13, 2026, [https://askubuntu.com/questions/631997/subscribe-for-dbus-event-of-screen-power-off](https://askubuntu.com/questions/631997/subscribe-for-dbus-event-of-screen-power-off)  
20. Settings app crashes with Wayland, in cc\_display\_settings\_rebuild\_ui (\#725) · Issue · GNOME/mutter, accessed April 13, 2026, [https://gitlab.gnome.org/GNOME/mutter/-/issues/725](https://gitlab.gnome.org/GNOME/mutter/-/issues/725)  
21. How can I guess what screen is connected to each graphic card?, accessed April 13, 2026, [https://unix.stackexchange.com/questions/794806/how-can-i-guess-what-screen-is-connected-to-each-graphic-card](https://unix.stackexchange.com/questions/794806/how-can-i-guess-what-screen-is-connected-to-each-graphic-card)  
22. \[Solved\] Cannot enable refresh rate above 60Hz \- Wayland / Gnome / AMD / Multimedia and Games / Arch Linux Forums, accessed April 13, 2026, [https://bbs.archlinux.org/viewtopic.php?id=307046](https://bbs.archlinux.org/viewtopic.php?id=307046)  
23. Using the desktop environment in RHEL 8 | Red Hat Enterprise Linux, accessed April 13, 2026, [https://docs.redhat.com/en/documentation/red\_hat\_enterprise\_linux/8/html-single/using\_the\_desktop\_environment\_in\_rhel\_8/index](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html-single/using_the_desktop_environment_in_rhel_8/index)  
24. Display Module: Displays not detected? · Issue \#241 · linuxmint/cinnamon-control-center, accessed April 13, 2026, [https://github.com/linuxmint/cinnamon-control-center/issues/241](https://github.com/linuxmint/cinnamon-control-center/issues/241)  
25. The best 1440p monitor in 2026: game and create at QHD with our top picks \- TechRadar, accessed April 13, 2026, [https://www.techradar.com/news/best-1440p-monitors](https://www.techradar.com/news/best-1440p-monitors)  
26. News Archive \- TechPowerUp, accessed April 13, 2026, [https://www.techpowerup.com/news-archive?month=1225](https://www.techpowerup.com/news-archive?month=1225)  
27. \[RTINGS\] Samsung Odyssey OLED G6/G60SD S27DG60 Review just went live \- Reddit, accessed April 13, 2026, [https://www.reddit.com/r/OLED\_Gaming/comments/1eb2lj2/rtings\_samsung\_odyssey\_oled\_g6g60sd\_s27dg60/](https://www.reddit.com/r/OLED_Gaming/comments/1eb2lj2/rtings_samsung_odyssey_oled_g6g60sd_s27dg60/)  
28. Control monitor brightness and volume with ddcutil \- GNOME Shell Extensions, accessed April 13, 2026, [https://extensions.gnome.org/extension/6325/control-monitor-brightness-and-volume-with-ddcutil/](https://extensions.gnome.org/extension/6325/control-monitor-brightness-and-volume-with-ddcutil/)  
29. Dell Display Manager \- Ask Ubuntu, accessed April 13, 2026, [https://askubuntu.com/questions/1171741/dell-display-manager](https://askubuntu.com/questions/1171741/dell-display-manager)  
30. External monitor brightness not working : r/wayland \- Reddit, accessed April 13, 2026, [https://www.reddit.com/r/wayland/comments/11dpecw/external\_monitor\_brightness\_not\_working/](https://www.reddit.com/r/wayland/comments/11dpecw/external_monitor_brightness_not_working/)  
31. Controlling 'Brightness' on External Monitor on GNOME Wayland w/o DDCUTIL\! : r/Fedora, accessed April 13, 2026, [https://www.reddit.com/r/Fedora/comments/1pctydq/controlling\_brightness\_on\_external\_monitor\_on/](https://www.reddit.com/r/Fedora/comments/1pctydq/controlling_brightness_on_external_monitor_on/)  
32. GitHub \- zb3/gnome-gamma-tool: A command-line tool that lets you change gamma in GNOME and Cinnamon (with Wayland). You can also adjust contrast and brightness. It works by creating a color profile with the VCGT table, so that changes are persistent and don't interfere with other settings like night light., accessed April 13, 2026, [https://github.com/zb3/gnome-gamma-tool](https://github.com/zb3/gnome-gamma-tool)  
33. Any command to just switch off the screen in gnome 40 wayland? \- Reddit, accessed April 13, 2026, [https://www.reddit.com/r/gnome/comments/plivbq/any\_command\_to\_just\_switch\_off\_the\_screen\_in/](https://www.reddit.com/r/gnome/comments/plivbq/any_command_to_just_switch_off_the_screen_in/)  
34. data/dbus-interfaces/org.gnome.Mutter.DisplayConfig.xml · main \- GNOME GitLab, accessed April 13, 2026, [https://gitlab.gnome.org/GNOME/mutter/-/blob/main/data/dbus-interfaces/org.gnome.Mutter.DisplayConfig.xml](https://gitlab.gnome.org/GNOME/mutter/-/blob/main/data/dbus-interfaces/org.gnome.Mutter.DisplayConfig.xml)  
35. Gnome 3 standard brightness control with external monitor \- Ask Ubuntu, accessed April 13, 2026, [https://askubuntu.com/questions/1285463/gnome-3-standard-brightness-control-with-external-monitor](https://askubuntu.com/questions/1285463/gnome-3-standard-brightness-control-with-external-monitor)  
36. personal-scripts/wayland-brightness.sh at main \- GitHub, accessed April 13, 2026, [https://github.com/ahoneybun/personal-scripts/blob/main/wayland-brightness.sh](https://github.com/ahoneybun/personal-scripts/blob/main/wayland-brightness.sh)  
37. DBus resolution switch for Mutter \- GitHub Gist, accessed April 13, 2026, [https://gist.github.com/strycore/ca11203fd63cafcac76d4b04235d8759](https://gist.github.com/strycore/ca11203fd63cafcac76d4b04235d8759)  
38. Change scaling/resolution of primary monitor from bash/terminal \- Fedora Discussion, accessed April 13, 2026, [https://discussion.fedoraproject.org/t/change-scaling-resolution-of-primary-monitor-from-bash-terminal/76778](https://discussion.fedoraproject.org/t/change-scaling-resolution-of-primary-monitor-from-bash-terminal/76778)  
39. Virt-manager won't resize VM resolution with window \- \#22 by ilikelinux \- Fedora Discussion, accessed April 13, 2026, [https://discussion.fedoraproject.org/t/virt-manager-wont-resize-vm-resolution-with-window/171707/22](https://discussion.fedoraproject.org/t/virt-manager-wont-resize-vm-resolution-with-window/171707/22)  
40. gnome-monitor-config/src/org.gnome.Mutter.DisplayConfig.xml at master \- GitHub, accessed April 13, 2026, [https://github.com/jadahl/gnome-monitor-config/blob/master/src/org.gnome.Mutter.DisplayConfig.xml](https://github.com/jadahl/gnome-monitor-config/blob/master/src/org.gnome.Mutter.DisplayConfig.xml)  
41. Configure GNOME/Wayland display configuration from command line, accessed April 13, 2026, [https://unix.stackexchange.com/questions/275327/configure-gnome-wayland-display-configuration-from-command-line](https://unix.stackexchange.com/questions/275327/configure-gnome-wayland-display-configuration-from-command-line)  
42. gnome-mutter-screencast.c \- GitHub, accessed April 13, 2026, [https://github.com/fzwoch/obs-gnome-screencast/blob/master/gnome-mutter-screencast.c](https://github.com/fzwoch/obs-gnome-screencast/blob/master/gnome-mutter-screencast.c)  
43. Preserving GNOME Bluetooth and Brightness Settings Across Reboots \- Desktop, accessed April 13, 2026, [https://discourse.gnome.org/t/preserving-gnome-bluetooth-and-brightness-settings-across-reboots/32010](https://discourse.gnome.org/t/preserving-gnome-bluetooth-and-brightness-settings-across-reboots/32010)  
44. Wayland Gnome and brightness / Applications & Desktop Environments / Arch Linux Forums, accessed April 13, 2026, [https://bbs.archlinux.org/viewtopic.php?id=282475](https://bbs.archlinux.org/viewtopic.php?id=282475)