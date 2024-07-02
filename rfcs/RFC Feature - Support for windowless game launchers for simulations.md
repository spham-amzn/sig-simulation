
# (Proposed) Windowless / Headless Game Launchers for Simulations

## Summary:

This RFC proposes introducing the ability to the client (game) launcher without the creation of a native window. Currently, the only launchers that do not create a client window are the server and headless server launchers. The feature to support windowless or headless client launchers is useful for simulation use cases since the native client windows is typically used just to provide visualization of the scene and is not used directly by any ROS2 sensor.

## What is the relevance of this feature?

The general use case for not creating a native client window when launching the client launcher is primarily useful for simulation scenarios and not of any particular value for gaming scenarios. Currently for simulation projects, the client window that is created provides a view based on a camera that is configured in the simulation scene being launched. The camera may or may not be tied directly to the simulation target (ie robot), and may just provide a view of the scene to provide photo-realistic view of the simulation in general. (In many cases, the camera view can be changed using the same mechanisms as games do).

There are scenarios where the client window is not needed all, mostly in cases where automation or CI/CD pipelines are used to validate robot code. Furthermore, the O3DE Atom renderer may or not be needed for simulation use cases that do not O3DE's high-quality visualizations at all and thus needing to only rely on the simulations Physics and non-camera ROS2 sensors. The renderer will be needed for scenarios where there are sensors that utilize the ROS2 camera sensor even if the client window is not necessary.



## Feature design description:


There are two additional client modes in this proposal that will be included to the current default client launcher behavior:

* **Window-less**
Window-less clients will still launch with rendering capabilities enabled, but will not launch any native client window to show the game/simulation scene based on its camera view. That means that the underlying RHI will still be initialized based on the current platform and hardware (DX12, Vulkan, etc). The launcher will still require video and graphics drivers to be available. The control of optionally creating the client window will be configured through the use of the [settings registry](https://www.docs.o3de.org/docs/user-guide/settings/).

* **Headless**
Headless clients will not launch any native client window, but will also will not have any rendering capabilities/support enabled. Similar to the headless server launcher, headless client launchers not initialize any graphic clients and disable any Atom RHI (null). (Refer to the [Headless Server Launcher RFC](https://github.com/o3de/sig-network/blob/main/rfcs/rfc-net-20230821-1-headless-server-launcher.md) ) Since this requires different build settings for the launcher in the same way the headless server launcher does, a new `headless game launcher` configuration will be added to the [unified launcher target](https://github.com/o3de/o3de/tree/development/Code/LauncherUnified).


## Technical design description:

### Client window creation control

#### Settings Registry
The control to create the client window in the client launcher will be controlled by the following settings registry key

```
/O3DE/Atom/Bootstrap/CreateNativeWindow
```

This key can be applied on a per-project case by providing a `bootstrap.setreg` file in the project's `Registry` subfolder. For example:

```json
{
  "O3DE": {
    "Atom": {
      "Bootstrap": {
        "CreateNativeWindow": false
      }
    }
  }
}
```
The default behavior of `CreateNativeWindow` will be `true`.

#### Atom Bootstrap

The creation of the native client window is triggered in the [BootstrapSystemComponent.cpp](https://github.com/o3de/o3de/blob/454b5625e55059955a0f35d84d62101e754bf7fd/Gems/Atom/Bootstrap/Code/Source/BootstrapSystemComponent.cpp#L327-L350) in the Atom gem. In addition to the check for the `appType`'s `IsHeadless()` attribute, the component will also read the settings registry key value for `CreateNativeWindow` and use this flag to also determine if `m_nativeWindow` is created or not.

### Headless launcher control

Similar to the [Headless Server Launcher](https://github.com/o3de/sig-network/blob/main/rfcs/rfc-net-20230821-1-headless-server-launcher.md), the headless client launcher will need to create specific headless launcher projects. Since the ground-work for removing the build dependencies for graphics related modules is already complete via the headless server update, the only update needed to introduce an additional `HeadlessGameLauncher` target to [launcher_generator.cmake](https://github.com/o3de/o3de/blob/stabilization/2409/Code/LauncherUnified/launcher_generator.cmake). 

**Details**

* A cmake global property `O3DE_GAME_LAUNCHER_TYPES` which will support `GameLauncher` and also `HeadlessGameLauncher` will be introduced. 
* The property will default to only `GameLauncher`, but can be overridden during project generation to either replace it with `HeadlessGameLauncher` or add it as an additonal target along with `GameLauncher`
* For `HeadGameLauncher`, it will implicitly disable the client window creation.
* For `HeadGameLauncher`, it will implicitly set Atom's rhi to `null`

## What are the advantages of the feature?

* Allows simulation clients to run on a non-graphics intensive environment (headless)
* Allows simulation clients to run on headless, server only operating systems
* Provides option to not show a client window on the launcher when not needed
* Disabling the main render pipeline that is used by the client will reduce the memory and resources used by the simulations that employs rendering from the ROS2 camera sensor.

## What are the disadvantages of the feature?

* Adds additional complexity to the unified launcher code

## How will this be implemented or integrated into the O3DE environment?

The changes needed will be backwards compatible with the current version of O3DE. 
* By default, if the `/O3DE/Atom/Bootstrap/CreateNativeWindow` is not set in the settings registry, the default behavior will create the client window for client launchers. 
* By default, only the `GameLauncher` launcher type will be enabled for the launcher project. Enabling headless client launcher will require explictly setting the `O3DE_GAME_LAUNCHER_TYPES` variable.

## Are there any alternatives to this feature?

This is partially supported currently by setting the `rhi` argument to `null` that can be passed into the launcher. However, this does not prevent the creation of a client window, it only disables the loading and initialization of the backend Atom RHI.

## How will users learn this feature?

This will be added to the o3de documentation, specifically the for the simulation and ROS2 specific topics.

## Are there any open questions?

TBD



