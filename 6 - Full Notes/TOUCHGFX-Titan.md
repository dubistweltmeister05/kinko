[[Tor]]
# TouchGFX Graphics Framework Design Document

## TorTitan Energy Meter – STM32H750VB Platform

---

## Document Information

| Field | Value |
|-------|-------|
| **Document Title** | TouchGFX Graphics Framework Design Document |
| **Project** | TorTitan Multi-Function Energy Meter |
| **Hardware Platform** | STM32H750VB (Cortex-M7, 480 MHz) |
| **Graphics Framework** | TouchGFX 4.25.0 |
| **Display Specification** | 320×480 TFT LCD, 16-bit RGB565 Color Depth |
| **User Input Method** | 5-way Joystick Navigation (Touchscreen Disabled) |
| **Document Version** | 1.0 |
| **Date** | May 2026 |
| **Status** | Release Candidate |

---

## Revision History

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | May 2026 | — | Initial release for knowledge transfer handover |

---

## Table of Contents

1. [TouchGFX Architecture and Project Flow](#1-touchgfx-architecture-and-project-flow)
2. [New Screen Generation and Navigation Flow](#2-new-screen-generation-and-navigation-flow)
3. [Symbols, Icons, and Asset Management](#3-symbols-icons-and-asset-management)
4. [Custom Containers and Reusable Components](#4-custom-containers-and-reusable-components)
5. [Interactions, Actions, Callbacks, and Event Handling](#5-interactions-actions-callbacks-and-event-handling)
6. [Font Generation and Typography Settings](#6-font-generation-and-typography-settings)
7. [Bits Per Pixel (BPP) Configuration](#7-bits-per-pixel-bpp-configuration)
8. [Wildcard Text Buffers](#8-wildcard-text-buffers)
9. [Widget Alignment and Layout](#9-widget-alignment-and-layout)
10. [Unicode Text Handling](#10-unicode-text-handling)
11. [Get/Set Methods and Data Flow Patterns](#11-getset-methods-and-data-flow-patterns)
12. [Presenter-Model-View Communication Architecture](#12-presenter-model-view-communication-architecture)
13. [Dynamic Text Update Handling](#13-dynamic-text-update-handling)
14. [VLN/VLL Waveform Graph Implementation](#14-vlnvll-waveform-graph-implementation)
15. [Power Quality and Harmonic Distortion Graphs](#15-power-quality-and-harmonic-distortion-graphs)
16. [Phasor Diagram Mathematics and Rendering](#16-phasor-diagram-mathematics-and-rendering)
17. [Canvas Widget and Vector Drawing](#17-canvas-widget-and-vector-drawing)
18. [Memory Optimization and Performance Tuning](#18-memory-optimization-and-performance-tuning)
19. [Common Issues and Debugging Procedures](#19-common-issues-and-debugging-procedures)
20. [Touch Controller Integration](#20-touch-controller-integration)
21. [Backend Integration Architecture](#21-backend-integration-architecture)
22. [Project-Specific Logic and Reusable Components](#22-project-specific-logic-and-reusable-components)
23. [Build Generation and Code Regeneration](#23-build-generation-and-code-regeneration)
24. [Critical Files and Overwrite Protection](#24-critical-files-and-overwrite-protection)

---

## 1. TouchGFX Architecture and Project Flow

### 1.1 Introduction and Purpose

The TorTitan Energy Meter employs STMicroelectronics' TouchGFX framework as its graphical user interface solution. TouchGFX is a highly optimized C++ graphics library specifically designed for STM32 microcontrollers, capable of achieving smooth 60 frames-per-second rendering on resource-constrained embedded platforms. The framework provides hardware-accelerated rendering through the STM32H7's ChromART (DMA2D) graphics accelerator, enabling sophisticated visual displays while maintaining low CPU overhead.

This document provides comprehensive technical documentation of the TouchGFX implementation within the TorTitan project, serving as both a design reference and knowledge transfer guide for developers who will maintain or extend the graphical interface.

### 1.2 System Architecture Overview

The graphical subsystem operates as an integral component within a FreeRTOS-based multi-tasking environment. The architecture establishes clear separation between the graphics rendering engine, the application logic layer, and the underlying energy meter firmware. Data flows from the energy measurement subsystem through a well-defined Model-View-Presenter (MVP) architecture, ultimately rendering on the 320×480 pixel TFT display.

The following diagram illustrates the high-level system architecture and the relationships between major software components:

```
┌──────────────────────────────────────────────────────────────────────┐
│                        STM32H750VB MCU                                │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────────────┐     │
│  │  FreeRTOS   │     │  TouchGFX   │     │   Energy Meter      │     │
│  │  Scheduler  │────▶│  Engine     │     │   Firmware (Core)   │     │
│  └─────────────┘     └──────┬──────┘     └──────────┬──────────┘     │
│                              │                       │                │
│                    ┌─────────▼─────────┐             │                │
│                    │   MVP Pattern     │             │                │
│                    │  Model ◀──────────┴─────────────┘                │
│                    │  Presenter                                       │
│                    │  View                                            │
│                    └─────────┬─────────┘                              │
│                              │                                        │
│                    ┌─────────▼─────────┐                              │
│                    │   FMC LCD Driver  │                              │
│                    │   + DMA2D Accel   │                              │
│                    └───────────────────┘                              │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

The STM32H750VB microcontroller serves as the central processing unit, running at 480 MHz and providing substantial computational headroom for both energy measurement calculations and graphics rendering. The FreeRTOS real-time operating system manages task scheduling, ensuring that the graphics subsystem receives consistent execution time while not interfering with time-critical measurement operations.

### 1.3 Project Directory Structure

The TouchGFX project follows a standardized directory hierarchy that separates user-modifiable code from auto-generated framework code. Understanding this structure is essential for effective development and maintenance, as modifications to generated files will be lost during subsequent code regeneration cycles.

```
TouchGFX/
├── gui/                        ← USER CODE (safe to edit)
│   ├── include/gui/
│   │   ├── model/              ← Model.hpp, ModelListener.hpp
│   │   ├── containers/         ← Custom container headers
│   │   └── <screen_name>/      ← View + Presenter headers per screen
│   └── src/
│       ├── model/              ← Model.cpp (backend bridge)
│       ├── containers/         ← Container implementations
│       └── <screen_name>/      ← View + Presenter .cpp files
├── generated/                  ← AUTO-GENERATED (DO NOT EDIT)
│   ├── gui_generated/          ← Base classes for all screens
│   ├── fonts/                  ← Compiled font data
│   ├── images/                 ← Compiled image assets
│   └── texts/                  ← String database
├── target/                     ← Hardware abstraction layer
│   ├── STM32TouchController.*  ← Touch driver (currently disabled)
│   ├── TouchGFXHAL.*          ← Display HAL implementation
│   └── generated/             ← Generated HAL (DO NOT EDIT)
├── assets/                     ← Raw design assets (images, fonts, texts)
├── App/                        ← Application entry point (app_touchgfx.c)
├── config/                     ← Build configuration (gcc/app.mk)
└── simulator/                  ← PC simulator configuration
```

The `gui/` directory contains all user-written application code and represents the primary location for development activities. Files within this directory persist across code regeneration operations and contain the custom logic that implements the energy meter's user interface behavior. The `generated/` directory, in contrast, contains auto-generated code produced by the TouchGFX Designer tool and must never be manually edited; any changes made here will be overwritten during the next code generation cycle.

### 1.4 Execution Flow and Frame Rendering

The TouchGFX engine operates on a frame-by-frame basis, synchronized to the display's vertical synchronization (VSYNC) signal. Each frame cycle follows a deterministic sequence of operations that polls for new data, processes user input, updates the display model, and renders changed regions to the framebuffer.

The execution sequence proceeds as follows: First, upon system boot, the `main.c` entry point initializes hardware peripherals and starts the FreeRTOS scheduler. The scheduler then activates the configured tasks, including the TouchGFX main task and the VSYNC signal task. The VSYNC task executes every 16 milliseconds (achieving approximately 60 frames per second) and signals the TouchGFX engine to begin a new frame cycle. During each frame, the TouchGFX task invokes `Model::tick()`, which polls the energy meter data structures and joystick input queue. Following data collection, the framework determines which screen regions require redrawing based on widget invalidation flags. Finally, the DMA2D hardware accelerator performs efficient block copy operations to transfer rendered content to the FMC-mapped LCD display memory.

```
main.c → Hardware Init → FreeRTOS Start
                              ↓
              ┌───────────────┴───────────────┐
              ↓                               ↓
        TouchGFX Task                    VSYNC Task
                                              │ (16ms)
                                              ↓
                                        Model::tick()
                                              ↓
                                     Poll Data & Events
                                              ↓
                                        Changed? ──No──→ Skip ─┐
                                          │Yes                 │
                                          ↓                    │
                                 Broadcast to Presenter        │
                                          ↓                    │
                                  View Updates Widgets         │
                                          ↓                    │
                                   Invalidate Changed          │
                                          ↓                    │
                                      DMA2D Render ←───────────┘
                                          ↓
                                      LCD Update
                                          │
                                    (Next Frame)
```

### 1.5 FreeRTOS Task Configuration

The graphics subsystem utilizes two dedicated FreeRTOS tasks, each configured with specific priority levels and stack allocations optimized for their respective responsibilities.

| Task Name | Priority | Stack Allocation | Functional Role |
|-----------|----------|------------------|-----------------|
| TouchGFX Main | Low (osPriorityLow) | 2176 bytes | Event loop processing and frame rendering |
| VSYNC Signal | Idle (osPriorityIdle) | 256 bytes | 16ms periodic tick signal generation |

The TouchGFX main task operates at low priority to ensure that higher-priority tasks (such as the energy measurement task) receive preferential CPU time. This prioritization scheme guarantees that measurement accuracy is never compromised by graphics rendering operations. The VSYNC task requires minimal stack space as it performs only signaling operations without complex data processing.

---

## 2. New Screen Generation and Navigation Flow

### 2.1 Screen Creation Process

The TouchGFX Designer application serves as the primary tool for creating new screens within the graphical interface. This visual design environment allows developers to compose screen layouts by placing and configuring widgets, while automatically generating the corresponding C++ source code. The generated code follows the Model-View-Presenter pattern, creating a clear separation between visual presentation and application logic.

To create a new screen, launch the TouchGFX Designer application by opening the project file located at `TorTitan_withStm32CubeIde.touchgfx`. Within the Designer interface, navigate to the Screens panel and click the "+" button to instantiate a new screen. Assign a descriptive name to the screen following the project's naming convention (e.g., `New_Feature_Screen`). The Designer canvas then becomes available for widget placement, where you can add text areas, images, containers, and other visual elements. After completing the visual design, initiate code generation by pressing F4 or clicking the Generate button in the toolbar.

### 2.2 Generated File Structure

The code generation process produces a carefully organized set of files that implement the screen's functionality. The framework generates base classes containing the widget hierarchy and default behavior, while also creating stub files for custom application logic that persist across regeneration cycles.

Upon code generation for a screen named `New_Feature_Screen`, the following file hierarchy is created:

```
generated/gui_generated/include/gui_generated/new_feature_screen_screen/
    New_Feature_ScreenViewBase.hpp     ← Base view class (DO NOT EDIT)
    New_Feature_ScreenPresenterBase.hpp ← Base presenter class (DO NOT EDIT)

generated/gui_generated/src/new_feature_screen_screen/
    New_Feature_ScreenViewBase.cpp     ← Base view implementation
    New_Feature_ScreenPresenterBase.cpp ← Base presenter implementation

gui/include/gui/new_feature_screen_screen/
    New_Feature_ScreenView.hpp         ← Custom view header (EDITABLE)
    New_Feature_ScreenPresenter.hpp    ← Custom presenter header (EDITABLE)

gui/src/new_feature_screen_screen/
    New_Feature_ScreenView.cpp         ← Custom view implementation (EDITABLE)
    New_Feature_ScreenPresenter.cpp    ← Custom presenter implementation (EDITABLE)
```

The base classes in the `generated/` directory contain all widget instantiation code, position definitions, and default event handlers defined within the Designer. These files are regenerated each time code generation is triggered and must never be manually modified. The derived classes in the `gui/` directory extend the base classes and provide locations for custom application logic. These files are created only once during the initial generation and remain untouched during subsequent regeneration operations.

### 2.3 Navigation Architecture

Screen navigation in the TorTitan project is implemented entirely through joystick input, as the touchscreen capability has been intentionally disabled. The navigation architecture routes joystick events through the Model layer, which broadcasts them to the currently active Presenter via the ModelListener interface. This design ensures that navigation behavior can be customized on a per-screen basis while maintaining a centralized input handling mechanism.

The following diagram shows the complete screen navigation map for the TorTitan energy meter interface:

```
Main Screens:
  VAF ←→ PMD ←→ ENG ←→ VLN ←→ VLL ←→ CURR ←→ PF ←→ PA

THD Screens:
  PA ←→ VTHD ←→ ITHD
  VTHD ──(Center)──→ VR_D ←→ VY_D ←→ VB_D
  ITHD ──(Center)──→ IR_D ←→ IY_D ←→ IB_D

Settings:
  VAF ──(Center)──→ Settings Menu → PTR, CTR, COMM, DT

System Transitions:
  Splash ──(Boot)──→ VAF
  Alarm ···(Trigger)···→ VAF
```

**Navigation Legend:**
- `←→` : Left/Right joystick navigation
- `⏺` : Center click (enter/select)
- Dashed lines: System-triggered transitions (alarms, sleep/wake)

The joystick event processing occurs within the `Model::tick()` function, which is invoked during each frame cycle. The implementation polls the joystick event queue and dispatches events to the appropriate listener methods:

```cpp
// Model.cpp - tick() function
Joystick_Event_t joy_event;
if (xQueueReceive(joystick_queue, &joy_event, 0) == pdTRUE) {
    switch (joy_event) {
        case JOYSTICK_LEFT:
            modelListener->handle_left_click();
            break;
        case JOYSTICK_RIGHT:
            modelListener->handle_right_click();
            break;
        case JOYSTICK_CENTER:
            modelListener->handle_center_click();
            break;
        case JOYSTICK_UP:
            modelListener->handle_up_click();
            break;
        case JOYSTICK_DOWN:
            modelListener->handle_down_click();
            break;
        case JOYSTICK_LONG_PRESS:
            modelListener->handle_long_press();
            break;
    }
}
```

Screen transitions employ the "NoTransition" variant of navigation, which performs instantaneous screen switches without animated transitions. This design decision was made to conserve memory and CPU resources, as animated transitions require double buffering and additional rendering overhead.

Navigation is triggered by calling custom actions (virtual functions) defined in the TouchGFX Designer. These actions are configured with interactions that perform the actual screen change internally. The `handle_xxx_click()` methods are implemented in the **View** class, where they call the custom action directly:

```cpp
// Example screen navigation from a View - calling custom actions defined in Designer
void VAFScreenView::handle_right_click() {
    takeToVoltageLN();  // Custom action triggers "Change screen" interaction
}

void VAFScreenView::handle_center_click() {
    takeToSettings();   // Custom action triggers "Change screen" interaction
}
```

### 2.4 Implementing Navigation for New Screens

When adding a new screen to the navigation flow, the process leverages TouchGFX Designer's interaction system. The framework wraps screen transitions inside custom actions that you define, providing a clean abstraction layer.

**Step 1: Define a Custom Action in TouchGFX Designer**

In the TouchGFX Designer, select the screen that will initiate navigation. In the screen's properties, define a new **Action** (virtual function):

- Navigate to the screen's properties panel
- Add a new action with a descriptive name using the `takeTo` prefix (e.g., `takeToNextScreen`, `takeToPreviousScreen`, `takeToSettings`)
- This creates a virtual function that can be called from code

**Step 2: Create an Interaction Using the Action as Trigger**

Create an interaction that connects your custom action to the screen transition:

- **Trigger:** Select the custom action you just defined (e.g., `takeToNextScreen`)
- **Action:** Select "Change screen" 
- **Target Screen:** Choose the destination screen
- **Transition:** Select "No Transition" (recommended for performance)

When the custom action is invoked, TouchGFX automatically executes the screen change.

**Step 3: Generate Code**

Press F4 to generate code. TouchGFX creates:
- A virtual function declaration in the View base class for your custom action
- Internal logic that calls the auto-generated `gotoXXXScreenNoTransition()` method when your action is triggered

**Step 4: Call Your Custom Action from Joystick Handlers**

In your View code, simply call the custom action you defined. The `handle_xxx_click()` methods are implemented in the View class:

```cpp
// In NewFeatureScreenView.cpp
void NewFeatureScreenView::handle_right_click() {
    // Call the custom action defined in Designer
    // TouchGFX handles the actual screen transition internally
    takeToNextScreen();
}

void NewFeatureScreenView::handle_left_click() {
    takeToPreviousScreen();
}
```

**Key Point:** You never directly call the auto-generated `application().gotoXXXScreenNoTransition()` methods. Instead, you call your custom action (virtual function) directly from the View, and the interaction system—configured in the Designer—handles the screen transition internally. This keeps navigation logic declarative and maintainable through the Designer interface.

---

## 3. Symbols, Icons, and Asset Management

### 3.1 Image Asset Pipeline

The TouchGFX framework processes image assets through a compilation pipeline that converts standard image formats into optimized binary representations suitable for embedded rendering. Source images are placed in the assets directory, processed by the Designer's code generator, and compiled into C arrays that reside in the microcontroller's flash memory.

Image files should be placed in the `TouchGFX/assets/images/` directory structure. The TouchGFX Designer automatically detects images within this directory and makes them available for use within screen designs. Supported formats include PNG (recommended for images requiring transparency), BMP (for simple images without transparency requirements), and JPEG (for photographic content, though rarely used in embedded applications due to decompression overhead).

### 3.2 Image Format Guidelines and Best Practices

Image assets should be prepared with careful attention to memory constraints and rendering performance. The following guidelines ensure optimal asset integration:

Color depth should match the project's configured bits-per-pixel (BPP) setting. This project operates at 16-bit color depth using the RGB565 format, meaning each pixel requires two bytes of storage. Images with higher color depth are automatically converted during compilation, but starting with appropriately formatted source images reduces processing time and potential quality loss.

Image dimensions should be kept as small as practical for the intended use case. Each pixel consumes two bytes in the RGB565 format, meaning a 100×100 pixel image requires 20,000 bytes of flash storage. Large images can quickly exhaust available flash memory and should be avoided when alternative approaches (such as canvas-based vector drawing) can achieve similar visual results.

File naming should follow a descriptive convention that indicates the image's purpose and usage context. Examples include `icon_wifi_connected.png`, `arrow_left_nav.png`, and `indicator_alarm_active.png`. This naming convention simplifies asset management and makes the codebase more maintainable.

### 3.3 Currently Implemented Icons

The TorTitan project utilizes a minimal set of icon assets, prioritizing memory efficiency by using simple geometric shapes and text labels wherever possible. The following icon categories are currently implemented:

Navigation indicators comprise arrow icons for directional guidance, indicating available navigation options (left, right, up, down) on various screens. WiFi and Bluetooth Low Energy (BLE) status indicators display connectivity states when these features are enabled. Phase direction symbols indicate the direction of power flow in applicable measurement contexts. Alarm indicators provide visual alerts when measurement thresholds are exceeded or system faults occur.

### 3.4 Asset Compilation and Access

During code generation, the TouchGFX Designer compiles image assets into C arrays stored within the `generated/images/` directory. Each image receives a unique identifier defined in `generated/images/include/BitmapDatabase.hpp`, which serves as the central registry for all compiled image assets.

To reference an image programmatically, use the bitmap identifier constants defined in the database:

```cpp
// Example: Setting an image widget's bitmap
myImageWidget.setBitmap(Bitmap(BITMAP_ICON_WIFI_CONNECTED_ID));
```

### 3.5 Alternative to Bitmap Images: Canvas Drawing

For simple geometric shapes such as lines, circles, rectangles, and arrows, the canvas widget system provides a memory-efficient alternative to bitmap images. Canvas widgets render vector graphics at runtime, consuming no flash memory for image storage. This approach is particularly valuable for dynamic visual elements that change based on runtime data, such as the phasor diagram vectors implemented in the Phase Angle screen.

The trade-off with canvas widgets involves CPU utilization: vector graphics require software rendering during each frame, whereas bitmap images are simply copied from flash memory using hardware-accelerated DMA2D transfers. This consideration influences the decision between canvas and bitmap approaches for specific visual elements.

---

## 4. Custom Containers and Reusable Components

### 4.1 Container Architecture Concepts

Custom containers represent one of TouchGFX's most powerful features for creating modular, reusable user interface components. A container encapsulates a collection of widgets along with associated logic into a self-contained unit that can be instantiated across multiple screens. This approach promotes code reuse, ensures visual consistency, and simplifies maintenance by centralizing common functionality.

From a software architecture perspective, containers behave similarly to screens but operate as embedded components rather than standalone views. Each container maintains its own widget hierarchy, can receive data through setter methods, and manages its own invalidation for efficient rendering. Containers do not participate directly in the MVP pattern; instead, they receive data from the View that contains them and focus purely on visual presentation.

### 4.2 Project Custom Container Inventory

The TorTitan project implements several custom containers that provide specialized visualization capabilities for energy meter data. Each container addresses a specific visualization requirement and can be reused across multiple screens where that visualization is needed.

| Container Name | Source File | Functional Purpose |
|----------------|-------------|-------------------|
| **Static_Graph** | `gui/src/containers/Static_Graph.cpp` | Three-phase waveform visualization for VLN, VLL, and current measurements |
| **Distortion_Graph** | `gui/src/containers/Distortion_Graph.cpp` | Horizontal bar chart displaying 16 harmonic orders |
| **Phase_Angle_Screens** | `gui/src/containers/Phase_Angle_Screens.cpp` | Phasor diagram with voltage and current vectors rendered using canvas widgets |
| **Distortion_Screen** | `gui/src/containers/Distortion_Screen.cpp` | Total Harmonic Distortion display with phase selection capability |
| **CustomContainer1/2/3** | `gui/src/containers/CustomContainer*.cpp` | Generic utility containers for common layout patterns |

The Static_Graph container deserves particular attention as it implements the core waveform visualization used throughout the voltage and current display screens. This container utilizes the TouchGFX Graph widget class with 64 data points representing one complete electrical cycle (0° to 360°). The container manages three separate graph elements for the R, Y, and B phases, each rendered with distinct colors to enable easy visual differentiation.

### 4.3 Creating New Custom Containers

The process of creating a new custom container begins in the TouchGFX Designer. Navigate to the Custom Containers tab in the Designer interface and click the "+" button to create a new container. Assign a descriptive name that reflects the container's purpose, then design the container's visual layout by adding widgets to the container canvas.

After completing the visual design, generate code to produce the container's base class and stub files. The generated structure mirrors the pattern used for screens, with base classes in the `generated/` directory and editable derived classes in the `gui/` directory.

Custom logic is implemented by overriding methods in the derived container class. Common patterns include data setter methods that populate widget content, setup methods that configure initial widget states, and event handlers that respond to user interaction:

```cpp
// Example container implementation pattern
class MyContainer : public MyContainerBase {
public:
    MyContainer();
    virtual ~MyContainer() {}
    
    virtual void initialize();
    
    // Custom data setter method
    void setData(float phase_r, float phase_y, float phase_b);
    
protected:
    // Internal helper methods
    void updatePhaseDisplay(int phase_index, float value);
};
```

### 4.4 Container-View Communication Pattern

When containers are placed on screens, they receive data through a well-defined communication chain that originates from the Model layer. The data flows through the Presenter to the View, which then invokes setter methods on the contained components. This unidirectional data flow ensures clear responsibility boundaries and simplifies debugging.

```
EM_PARAMS / Waveforms
        ↓
   Model::tick()
        ↓
    Presenter
        ↓
      View
        ↓
   Containers (Static_Graph / Distortion_Graph / Phasor)
        ↓
   Widgets (Graphs / Bars / Lines)
        ↓
   DMA2D Render
```

The View implementation is responsible for extracting relevant data from the received structure and passing it to the appropriate container methods:

```cpp
void Voltage_LNView::set_voltage_data(voltage_data_t data) {
    // Pass waveform data to the graph container
    waveformGraph.set_graph_data(data.vr_waveform, 
                                  data.vy_waveform, 
                                  data.vb_waveform);
    
    // Update other widgets with scalar values
    // ...
}
```

---

## 5. Interactions, Actions, Callbacks, and Event Handling

### 5.1 Event Handling Architecture Overview

The event handling architecture in the TorTitan project differs significantly from typical TouchGFX applications due to the exclusive use of joystick navigation rather than touchscreen input. While TouchGFX Designer provides comprehensive support for touch-based interactions (button clicks, swipe gestures, etc.), this project routes all user input through the joystick subsystem, which integrates with TouchGFX via the Model layer.

The event architecture operates on a publish-subscribe pattern implemented through the ModelListener interface. The Model layer publishes events (joystick actions, alarm triggers, data updates) to the currently active Presenter, which subscribes to these events by implementing the ModelListener interface. This design allows each screen to define custom responses to input events while maintaining centralized event generation.

### 5.2 TouchGFX Designer Interactions

Although the project primarily uses code-based event handling, the TouchGFX Designer provides interaction configuration capabilities that can be useful for certain scenarios. Designer interactions define trigger-action pairs: when a specified trigger occurs (button clicked, screen entered, animation completed), the framework executes the associated action (change screen, call virtual function, modify widget property).

For screens that do utilize Designer-configured interactions, the interaction definitions are compiled into the generated base class code. Common interaction types include:

**Screen Entry Triggers:** Execute actions when a screen becomes active, useful for initializing display states or starting animations.

**Virtual Function Calls:** Invoke methods defined in the derived View class, enabling custom logic execution in response to widget events.

**Property Modifications:** Change widget properties such as visibility, position, or content in response to triggers.

### 5.3 Joystick Event Callback Implementation

The primary event handling mechanism in this project centers on the joystick callback system. The ModelListener interface defines virtual methods for each joystick event type, which Presenters override to implement screen-specific behavior.

The following diagram illustrates the complete joystick event flow from hardware interrupt to screen navigation:

```
Joystick GPIO → ISR → Queue → Model::tick() → Presenter → View.handle_xxx_click() → takeToXXX() → New Screen
                                    (L/R/U/D/C)
```

The ModelListener interface establishes the complete set of supported events:

```cpp
// ModelListener.hpp - Complete event interface definition
class ModelListener {
public:
    virtual ~ModelListener() {}
    
    // Joystick navigation events
    virtual void handle_left_click() {}
    virtual void handle_right_click() {}
    virtual void handle_up_click() {}
    virtual void handle_down_click() {}
    virtual void handle_center_click() {}
    virtual void handle_long_press() {}
    
    // System events
    virtual void trigger_alarm_screen() {}
    virtual void back_to_VAF() {}
    
    // Data update notifications
    virtual void set_vaf_data(vaf_data_t data) {}
    virtual void set_voltage_ln_data(voltage_data_t data) {}
    // ... additional data setters for each screen type
};
```

**View implementations** override these methods to define screen-specific behavior. A typical implementation pattern involves navigation on left/right events and action execution on center click. The `handle_xxx_click()` methods are implemented in the View class, calling custom actions directly:

```cpp
// VAFScreenView.cpp - Example implementation
void VAFScreenView::handle_right_click() {
    // Call custom action defined in Designer - triggers screen change interaction
    takeToVoltageLN();
}

void VAFScreenView::handle_left_click() {
    // Call custom action defined in Designer - triggers screen change interaction
    takeToAbout();
}

void VAFScreenView::handle_center_click() {
    // Call custom action defined in Designer - triggers screen change interaction
    takeToSettings();
}
```

**Note:** The action names (`takeToVoltageLN`, `takeToAbout`, `takeToSettings`) are defined in the TouchGFX Designer using the `takeTo` prefix convention. Each action has an associated interaction that specifies the target screen and transition type.

### 5.4 Alarm Interrupt Handling

The alarm system demonstrates an important pattern for handling asynchronous events that can interrupt normal user interaction. When measurement values exceed configured thresholds, the energy meter firmware sets the `trigger_alarm` flag. The Model layer monitors this flag and broadcasts alarm events to the active Presenter, which can then initiate navigation to the alarm display screen.

The alarm detection logic implements edge detection to prevent repeated triggers on a sustained alarm condition:

```cpp
// Model.cpp - Alarm interrupt handling in tick()
if (trigger_alarm != prev_trigger_alarm) {
    prev_trigger_alarm = trigger_alarm;
    if (trigger_alarm) {
        // Alarm condition just became active
        modelListener->trigger_alarm_screen();
    } else {
        // Alarm condition just cleared
        modelListener->back_to_VAF();
    }
}
```

### 5.5 Auto-Scroll Feature Implementation

The auto-scroll feature provides automatic screen cycling for demonstration or monitoring scenarios where manual navigation is impractical. When enabled, the system automatically triggers right-navigation events at configurable intervals, cycling through the measurement screens.

The implementation uses a frame counter that accumulates during each `Model::tick()` invocation. When the counter exceeds the configured threshold, the system simulates a right-click event and resets the counter:

```cpp
// Model.cpp - Auto-scroll logic
if (auto_scroll_enabled && current_screen_supports_auto_scroll) {
    tick_count++;
    if (tick_count >= auto_scroll_interval_frames) {
        modelListener->handle_right_click();
        tick_count = 0;
    }
} else {
    tick_count = 0;  // Reset counter when disabled
}
```

The auto-scroll interval is user-configurable through the Settings screens and persists across power cycles via non-volatile storage.

---

## 6. Font Generation and Typography Settings

### 6.1 Typography System Overview

The TouchGFX typography system manages font rendering for all text elements within the user interface. Unlike desktop systems that can dynamically load fonts from the filesystem, embedded applications must compile fonts into the firmware image. This compilation process converts TrueType or OpenType font files into optimized bitmap representations that the rendering engine can efficiently display.

The typography configuration significantly impacts both visual quality and memory consumption. Each font size and style combination requires separate compiled data, and the character set included in each typography directly determines storage requirements. Careful typography planning is essential to balance visual design requirements against the memory constraints of the target platform.

### 6.2 Configured Typographies

The TorTitan project utilizes two primary font families selected for their readability on the 320×480 pixel display and their licensing compatibility with commercial embedded products.

| Font Family | Available Sizes | Style | Primary Usage |
|-------------|-----------------|-------|---------------|
| **Lato Bold** | 12, 14, 16, 20, 24, 28, 40, 48 pixels | Bold | Measurement values, screen headers, emphasized labels |
| **Lato Regular** | 15, 16, 18 pixels | Regular | Body text, descriptive labels, secondary information |
| **Verdana** | 8, 20, 40 pixels | Regular | Numeric displays, small annotations, tabular data |

The Lato font family provides excellent readability at both large and small sizes, making it suitable for the diverse text elements present in the energy meter interface. The Verdana font serves as a secondary option, particularly useful for numeric data display where its wide character spacing improves digit differentiation.

### 6.3 Font Configuration Parameters

Typography configuration occurs within the TouchGFX Designer under the Texts → Typographies section. Each typography definition specifies several parameters that affect rendering quality and memory usage:

**Font Family and Size:** Specifies the source font file and the target rendering size in pixels. The rendering size determines the final appearance and cannot be changed at runtime.

**Font BPP (Bits Per Pixel for Anti-aliasing):** Controls the anti-aliasing quality of rendered glyphs. This project uses 4-bit anti-aliasing, providing 16 levels of transparency blending that produces smooth character edges while consuming half the memory of 8-bit anti-aliasing. Note that font BPP is independent of display BPP; a 16-bit color display can render fonts with 4-bit anti-aliasing.

**Kerning:** Enables or disables kerning tables for the typography. Kerning adjusts spacing between specific character pairs (such as "AV" or "To") to improve visual appearance. All typographies in this project have kerning enabled.

**Character Range:** Defines which Unicode characters are included in the compiled typography. Only specified characters are compiled; displaying unincluded characters results in blank spaces. The character range should include all characters that may appear in dynamic text, including digits, letters, unit symbols (V, A, W, kWh), and special characters (°, %, ±).

**Text Cache Size:** The project allocates 4096 bytes for the text rendering cache, which stores recently rendered text fragments to avoid redundant glyph rendering.

### 6.4 Adding New Fonts or Typographies

To add a new font to the project, place the font file (TrueType `.ttf` or OpenType `.otf` format) in the `TouchGFX/assets/fonts/` directory. Open the TouchGFX Designer and navigate to Texts → Typographies. Click the "+" button to create a new typography definition, then configure the parameters described above.

After configuring the typography, associate it with text elements that should use the new font. This association is made either through the Designer interface (for static text) or through the Texts tab where text resources are defined. Generate code to compile the new typography into the firmware image.

### 6.5 Memory Considerations

Font data represents a significant portion of flash memory consumption in most TouchGFX applications. Memory usage scales with font size (larger fonts require more pixels per glyph), character count (each included character requires storage), and BPP setting (higher BPP requires more bits per pixel).

As a guideline, a 48-pixel Lato Bold typography with the extended Latin character set may consume 50-100 KB of flash memory, while an 8-pixel Verdana typography with digits only may require only 2-5 KB. Monitor flash memory usage after adding new typographies and consider removing unused font sizes to optimize memory utilization.

---

## 7. Bits Per Pixel (BPP) Configuration

### 7.1 Display Color Depth

The bits-per-pixel (BPP) setting determines the color depth of the display and directly impacts both visual quality and memory requirements. The TorTitan project operates at 16-bit color depth using the RGB565 format, which provides 65,536 distinct colors while maintaining reasonable memory efficiency.

In the RGB565 format, each pixel is represented by a 16-bit value with the following bit allocation: 5 bits for the red channel (32 levels), 6 bits for the green channel (64 levels), and 5 bits for the blue channel (32 levels). The additional bit for green reflects the human eye's greater sensitivity to green wavelengths, providing perceptually better color reproduction than an even 5-5-5 distribution.

### 7.2 BPP Impact Analysis

The following table summarizes available BPP options and their characteristics:

| BPP | Color Count | Memory Per Pixel | Typical Application |
|-----|-------------|------------------|---------------------|
| 1 | 2 | 1 bit | Monochrome displays |
| 4 | 16 | 4 bits | Low-color e-ink displays |
| 8 | 256 | 1 byte | Simple LCD displays |
| **16** | **65,536** | **2 bytes** | **TorTitan project (RGB565)** |
| 24 | 16.7 million | 3 bytes | Full-color displays |
| 32 | 16.7 million + alpha | 4 bytes | Displays requiring transparency |

The 16-bit configuration represents an optimal balance for this application. The 65,536 available colors are sufficient for rendering charts, graphs, and measurement displays with good visual quality, while the 2-byte-per-pixel requirement keeps framebuffer memory consumption manageable.

### 7.3 Framebuffer Memory Requirements

The framebuffer stores the rendered display content and must be large enough to contain one complete frame. For the TorTitan display configuration, the framebuffer requirement calculates as:

```
Framebuffer Size = Width × Height × Bytes Per Pixel
                 = 320 × 480 × 2
                 = 307,200 bytes (approximately 300 KB)
```

This substantial memory requirement influences the decision to use single buffering rather than double buffering in this project. Double buffering would require 600 KB for framebuffers alone, which would exceed the available internal RAM of the STM32H750VB.

### 7.4 BPP in the Build System

The BPP configuration is specified in the build system to ensure consistent compilation across all project components. The setting appears in the GCC makefile at `TouchGFX/config/gcc/app.mk`:

```makefile
-DUSE_BPP=16
```

This preprocessor definition propagates through the build to configure TouchGFX internal data structures, image conversion routines, and rendering algorithms for 16-bit operation.

### 7.5 Font BPP vs. Display BPP

It is important to distinguish between display BPP and font BPP, as these are independent settings that serve different purposes. Display BPP (16-bit in this project) determines how pixel colors are stored in the framebuffer and sent to the display hardware. Font BPP (4-bit in this project) determines the anti-aliasing quality of rendered text glyphs.

Font anti-aliasing works by storing transparency information for each pixel within a glyph. A 4-bit font provides 16 transparency levels, allowing the rendering engine to blend text smoothly against background colors. When rendering text, the TouchGFX engine reads the 4-bit anti-aliasing data and composites it with the text color and background to produce final 16-bit pixel values in the framebuffer.

---

## 8. Wildcard Text Buffers

### 8.1 Wildcard Concept and Purpose

Wildcards represent TouchGFX's mechanism for displaying dynamic text content within TextArea widgets. While static text is defined at design time and compiled into the text database, wildcard text can be modified at runtime to display changing values such as measurement readings, configuration parameters, or status information.

Each wildcard consists of a Unicode character buffer associated with a TextArea widget. The application writes formatted text into the buffer, then signals the widget to redraw with the updated content. This approach separates the data formatting logic (implemented in View code) from the widget configuration (defined in the Designer).

### 8.2 Wildcard Configuration in TouchGFX Designer

Wildcards are configured through the TouchGFX Designer interface when editing TextArea widgets. The configuration process involves the following steps:

Select a TextArea widget and access its properties panel. Enable the wildcard feature by specifying a buffer for Wildcard 1 (and optionally Wildcard 2 for text areas requiring multiple dynamic segments). Specify the buffer size in Unicode characters, which must accommodate the longest possible string plus one character for null termination. Optionally provide initial text that appears before the application updates the buffer.

The Designer generates buffer declarations in the base View class header file, making them accessible to the derived View implementation where update logic resides.

### 8.3 Wildcard Update Pattern

The TorTitan project follows a consistent pattern for wildcard updates across all measurement screens. This pattern ensures proper data formatting, Unicode conversion, and widget invalidation:

```cpp
// Standard wildcard update implementation pattern
void Voltage_LNView::set_voltage_data(voltage_data_t data) {
    char temp[50];  // Temporary buffer for C-string formatting
    
    // Format the numeric value with appropriate precision
    snprintf(temp, sizeof(temp), "%.2f", data.vr);
    
    // Convert the UTF-8 C-string to TouchGFX Unicode format
    Unicode::fromUTF8((uint8_t*)temp, vr_valueBuffer, VR_VALUE_SIZE);
    
    // Signal the widget to redraw with updated content
    vr_text.invalidate();
}
```

This three-step pattern—format, convert, invalidate—appears throughout the codebase and should be followed when implementing new wildcard updates.

### 8.4 Buffer Size Planning

Buffer size selection requires careful consideration of the maximum possible string length. Undersized buffers result in truncated display, while oversized buffers waste RAM. The following guidelines inform buffer size decisions:

For numeric values, consider the maximum magnitude and precision. A voltage reading displayed as "12345.67" requires 8 characters plus null terminator, suggesting a minimum buffer size of 9. Adding margin for unexpected conditions or format changes, a buffer size of 12-15 characters provides adequate safety.

For values with unit prefixes (auto-scaling from V to kV to MV), include space for the prefix and unit characters. A complete formatted string like "123.45 MV" requires 9 characters plus null terminator.

For status text or descriptive strings, enumerate all possible values and size the buffer for the longest option plus null terminator.

### 8.5 Common Wildcard Patterns

The project employs several standardized format patterns for different data types:

| Data Category | Format String | Example Output | Typical Buffer Size |
|---------------|---------------|----------------|---------------------|
| Voltage | `"%.2f"` | "230.45" | 12 characters |
| Current | `"%.3f"` | "5.123" | 12 characters |
| Power | `"%.1f"` | "1250.5" | 12 characters |
| Percentage | `"%.1f%%"` | "4.5%" | 10 characters |
| Integer | `"%d"` | "150" | 8 characters |
| Invalid Data | `"--"` | "--" | 4 characters |

The invalid data indicator ("--") appears when measurement values contain NaN (Not a Number) or infinity values, indicating sensor errors or calculation issues.

---

## 9. Widget Alignment and Layout

### 9.1 Text Alignment Principles

Text alignment within TextArea widgets controls how text content positions itself within the widget's bounding rectangle. Proper alignment ensures visual consistency across screens and enables readable presentation of numeric data in tabular formats.

TouchGFX supports three horizontal alignment modes, each suitable for different content types:

**Left Alignment** positions text at the left edge of the widget boundary, allowing text to flow toward the right. This alignment is the default for most label text and is appropriate for descriptive content where reading naturally proceeds left-to-right.

**Center Alignment** positions text equidistant from both horizontal edges, creating a balanced appearance suitable for headers, titles, and single-line value displays where the containing widget provides visual framing.

**Right Alignment** positions text at the right edge of the widget boundary, with content extending toward the left. This alignment is essential for numeric values in columnar displays, ensuring decimal points align vertically across multiple rows.

### 9.2 Widget Positioning and Layout

The TouchGFX Designer provides comprehensive tools for precise widget positioning. Widgets can be placed using absolute pixel coordinates, aligned relative to parent containers, or positioned using snap-to-grid functionality for consistent spacing.

Container-relative positioning becomes particularly important when designing reusable custom containers. Widgets within containers use coordinates relative to the container's origin (top-left corner), allowing the container to be placed anywhere on a screen while maintaining internal layout integrity.

The Designer stores position data in the `.touchgfx` project file as absolute pixel coordinates. These coordinates compile into the generated base class code, where widget position methods are called during screen initialization.

### 9.3 Auto-Size vs. Fixed Width Text Areas

TextArea widgets offer two sizing modes that affect how the widget responds to content changes:

**Auto-size mode** causes the widget to automatically adjust its dimensions to fit the text content. This mode is appropriate for static labels where the text is known at design time and will not change during runtime. Auto-sized widgets provide exact-fit bounding that can simplify layout calculations.

**Fixed-width mode** maintains constant widget dimensions regardless of content length. This mode is mandatory for wildcard text areas displaying dynamic values, as it prevents layout shifts when values change. Fixed-width text areas may truncate content that exceeds the allocated space, making proper buffer size planning essential.

### 9.4 Alignment Best Practices for Measurement Displays

The TorTitan interface displays numerous measurement values that benefit from consistent alignment strategies:

For wildcard values displaying numeric measurements, use fixed-width text areas with right alignment. This configuration ensures that decimal points align vertically when multiple values are displayed in columnar format, improving readability and enabling quick visual comparison between values.

For unit labels adjacent to values (V, A, W, etc.), use left-aligned fixed-width text areas positioned immediately after the value widgets. This separation allows unit labels to remain stationary while values update, preventing visual disturbance.

For screen headers and titles, use center-aligned text areas spanning the full display width. This creates clear visual hierarchy and provides balanced screen composition.

When displaying tabular data with multiple columns, consider using the Verdana font family, which provides more consistent character widths than proportional fonts like Lato. Consistent character widths improve column alignment without requiring complex width calculations.

---

## 10. Unicode Text Handling

### 10.1 Unicode Architecture in TouchGFX

TouchGFX employs 16-bit Unicode characters (UTF-16 encoding) for internal text representation. This encoding supports the vast majority of characters in worldwide writing systems while maintaining memory efficiency compared to 32-bit encodings. All text processing within the TouchGFX framework—including rendering, text measurement, and string manipulation—operates on this 16-bit character type.

The Unicode architecture requires explicit conversion when working with standard C strings, which typically use 8-bit UTF-8 encoding. The TouchGFX framework provides conversion utilities that bridge between these encodings, enabling integration with standard C library functions for formatting while maintaining compatibility with the rendering engine.

### 10.2 Conversion Pipeline for Dynamic Text

When updating wildcard text buffers with runtime data, the implementation must convert between C-string format (used for standard formatting functions) and TouchGFX Unicode format (required for display). The conversion pipeline follows a consistent sequence:

```cpp
// Step 1: Format data using standard C library functions
char temp[50];
snprintf(temp, sizeof(temp), "%.2f", value);  // Produces UTF-8 C-string

// Step 2: Convert UTF-8 to TouchGFX Unicode
Unicode::fromUTF8((uint8_t*)temp, destinationBuffer, BUFFER_SIZE);
// First parameter: source UTF-8 string cast to uint8_t pointer
// Second parameter: destination Unicode buffer
// Third parameter: maximum characters to write (prevents overflow)

// Step 3: Invalidate widget to trigger redraw
myTextArea.invalidate();
```

This pattern appears consistently throughout the View implementations in the TorTitan project. Developers adding new dynamic text displays should follow this established pattern to ensure consistency and prevent common errors.

### 10.3 Unicode Utility Functions

The TouchGFX Unicode namespace provides essential utility functions for string operations:

```cpp
// Convert UTF-8 encoded string to Unicode buffer
Unicode::fromUTF8(const uint8_t* utf8, UnicodeChar* dst, uint16_t maxChars);

// Format directly to Unicode buffer (similar to snprintf)
Unicode::snprintf(UnicodeChar* dst, uint16_t dstSize, const char* format, ...);

// Determine Unicode string length (character count, not byte count)
uint16_t Unicode::strlen(const UnicodeChar* str);

// Copy Unicode string with length limit
Unicode::strncpy(UnicodeChar* dst, const UnicodeChar* src, uint16_t maxChars);
```

The `Unicode::snprintf` function can format data directly to Unicode buffers, bypassing the intermediate C-string step. However, the TorTitan project primarily uses the two-step approach (snprintf to C-string, then fromUTF8 conversion) because it provides clearer debugging visibility and follows patterns familiar to C developers.

### 10.4 Critical Unicode Handling Rules

Several rules must be followed to prevent text rendering issues and memory corruption:

**Null Termination Requirement:** All Unicode buffers must be null-terminated. The conversion functions handle termination automatically when provided accurate buffer sizes, but manual buffer manipulation must explicitly add the null terminator.

**Direct C-String Usage Prohibition:** Never assign raw `char*` strings directly to Unicode buffers or TextArea widgets. The character types are incompatible (8-bit vs. 16-bit), and such assignments produce garbled display or memory corruption. Always use `Unicode::fromUTF8()` for conversion.

**Buffer Size Parameters:** Always provide accurate maximum size parameters to prevent buffer overflows. Unicode buffers are sized in characters (not bytes), so a 20-character buffer occupies 40 bytes of memory.

**Font Character Range:** Characters displayed through wildcards must be included in the typography's character range defined in the Designer. If a character is not compiled into the font, it renders as blank space without generating any runtime error.

### 10.5 Special Character Handling

The TorTitan interface displays various unit symbols and special characters that require attention during typography configuration:

Standard unit characters (V, A, W, Hz) are typically included in basic Latin character ranges. Derived unit characters (kWh, VAR) require lowercase letters (k, h) and uppercase letters (V, A, R) to be included. The degree symbol (°) for temperature or angle display requires explicit inclusion in the character range. Percentage symbol (%), plus/minus (±), and other mathematical symbols must be explicitly added when needed.

When adding new display elements that require special characters, verify that the typography includes the necessary characters by checking the Typographies configuration in the TouchGFX Designer.

---

## 11. Get/Set Methods and Data Flow Patterns

### 11.1 Unidirectional Data Flow Architecture

The TorTitan application implements a unidirectional data flow architecture where measurement data flows downstream from the firmware layer through the graphics stack to the display, while configuration changes flow upstream from user interface interactions to the firmware layer. This separation simplifies reasoning about data dependencies and prevents circular update patterns that could cause rendering issues.

### 11.2 Complete Data Flow Diagram with Code Examples

The following comprehensive diagram illustrates both downstream (measurement data) and upstream (configuration) data flows, showing the exact files, methods, and code patterns involved:

```
╔══════════════════════════════════════════════════════════════════════════════════════════════════════╗
║                           DOWNSTREAM DATA FLOW (Measurement Display)                                  ║
╠══════════════════════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                                       ║
║  ┌─────────────────────────────┐                                                                      ║
║  │   FIRMWARE LAYER            │                                                                      ║
║  │   File: Core/Src/EM.c       │                                                                      ║
║  │                             │                                                                      ║
║  │   extern PARAMS_t EM_PARAMS;│  ◄── Shared data structure updated by measurement task              ║
║  │   EM_PARAMS.Vpn = 230.5f;   │                                                                      ║
║  │   EM_PARAMS.Freq = 50.02f;  │                                                                      ║
║  │   EM_PARAMS.Iavg = 5.25f;   │                                                                      ║
║  └──────────────┬──────────────┘                                                                      ║
║                 │ (extern access)                                                                     ║
║                 ▼                                                                                     ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐                      ║
║  │   MODEL LAYER                                                               │                      ║
║  │   File: TouchGFX/gui/src/model/Model.cpp                                    │                      ║
║  │                                                                             │                      ║
║  │   void Model::tick() {                                                      │                      ║
║  │       if (data_update_counter >= 30) {        // Throttle to ~500ms         │                      ║
║  │           vaf_data.voltage_ltn = EM_PARAMS.Vpn;                             │                      ║
║  │           vaf_data.frequency = EM_PARAMS.Freq;                              │                      ║
║  │           modelListener->set_vaf_data(vaf_data);  // BROADCAST via getter   │                      ║
║  │       }                                                                     │                      ║
║  │   }                                                                         │                      ║
║  └──────────────┬──────────────────────────────────────────────────────────────┘                      ║
║                 │ modelListener->set_vaf_data(data)                                                   ║
║                 ▼                                                                                     ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐                      ║
║  │   PRESENTER LAYER                                                           │                      ║
║  │   File: TouchGFX/gui/src/vaf_screen/VAFPresenter.cpp                        │                      ║
║  │                                                                             │                      ║
║  │   void VAFPresenter::set_vaf_data(vaf_data_t data) {                        │                      ║
║  │       getView().set_vaf_data(data);  // Forward to View via SETTER          │                      ║
║  │   }                                                                         │                      ║
║  └──────────────┬──────────────────────────────────────────────────────────────┘                      ║
║                 │ getView().set_vaf_data(data)                                                        ║
║                 ▼                                                                                     ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐                      ║
║  │   VIEW LAYER                                                                │                      ║
║  │   File: TouchGFX/gui/src/vaf_screen/VAFView.cpp                             │                      ║
║  │                                                                             │                      ║
║  │   void VAFView::set_vaf_data(vaf_data_t data) {                             │                      ║
║  │       // Validate                                                           │                      ║
║  │       if (isnan(data.voltage_ltn)) { display "--"; return; }                │                      ║
║  │                                                                             │                      ║
║  │       // Format with auto-scaling                                           │                      ║
║  │       char temp[50];                                                        │                      ║
║  │       snprintf(temp, sizeof(temp), "%.1f", data.voltage_ltn);               │                      ║
║  │       Unicode::fromUTF8((uint8_t*)temp, voltage_buffer, BUFFER_SIZE);       │                      ║
║  │                                                                             │                      ║
║  │       // Trigger redraw                                                     │                      ║
║  │       voltage_text.invalidate();                                            │                      ║
║  │   }                                                                         │                      ║
║  └──────────────┬──────────────────────────────────────────────────────────────┘                      ║
║                 │ widget.invalidate()                                                                 ║
║                 ▼                                                                                     ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐                      ║
║  │   RENDERING                                                                 │                      ║
║  │   DMA2D hardware renders invalidated regions to LCD framebuffer             │                      ║
║  └─────────────────────────────────────────────────────────────────────────────┘                      ║
║                                                                                                       ║
╠══════════════════════════════════════════════════════════════════════════════════════════════════════╣
║                           UPSTREAM DATA FLOW (Configuration Save)                                     ║
╠══════════════════════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                                       ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐                      ║
║  │   VIEW LAYER (User Input Capture)                                           │                      ║
║  │   File: TouchGFX/gui/src/settings_pt_ratio_screen/Settings_PT_RatioView.cpp │                      ║
║  │                                                                             │                      ║
║  │   void Settings_PT_RatioView::handle_center_click() {                       │                      ║
║  │       uint32_t primary = get_entered_primary();     // GET from UI          │                      ║
║  │       uint32_t secondary = get_entered_secondary(); // GET from UI          │                      ║
║  │       presenter->save_pt_ratio(primary, secondary); // Forward to Presenter │                      ║
║  │   }                                                                         │                      ║
║  └──────────────┬──────────────────────────────────────────────────────────────┘                      ║
║                 │ presenter->save_pt_ratio(primary, secondary)                                        ║
║                 ▼                                                                                     ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐                      ║
║  │   PRESENTER LAYER (Validation & Persistence)                                │                      ║
║  │   File: TouchGFX/gui/src/settings_pt_ratio_screen/Settings_PT_RatioPresenter│                      ║
║  │                                                                             │                      ║
║  │   void Settings_PT_RatioPresenter::save_pt_ratio(uint32_t p, uint32_t s) {  │                      ║
║  │       EM_CONFIG.PT_Primary = p;      // SET config structure                │                      ║
║  │       EM_CONFIG.PT_Secondary = s;    // SET config structure                │                      ║
║  │       nv_storage_save(&EM_CONFIG);   // Persist to flash                    │                      ║
║  │       getView().takeToSaveSuccess(); // Navigate to confirmation            │                      ║
║  │   }                                                                         │                      ║
║  └──────────────┬──────────────────────────────────────────────────────────────┘                      ║
║                 │ nv_storage_save(&EM_CONFIG)                                                         ║
║                 ▼                                                                                     ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐                      ║
║  │   STORAGE LAYER                                                             │                      ║
║  │   File: Core/Src/nv_storage.c                                               │                      ║
║  │                                                                             │                      ║
║  │   void nv_storage_save(EM_CONFIG_t* config) {                               │                      ║
║  │       w25q64_write_sector(CONFIG_SECTOR, config, sizeof(EM_CONFIG_t));      │                      ║
║  │   }                                                                         │                      ║
║  └─────────────────────────────────────────────────────────────────────────────┘                      ║
║                                                                                                       ║
╚══════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

### 11.3 Key Get/Set Method Patterns

| Pattern | Direction | Example Method | File Location |
|---------|-----------|----------------|---------------|
| Data Getter | Firmware→Model | `EM_PARAMS.Vpn` | Model.cpp reads from EM.c |
| Data Setter | Model→Presenter | `modelListener->set_vaf_data()` | Model.cpp calls ModelListener |
| View Setter | Presenter→View | `getView().set_vaf_data()` | Presenter.cpp calls View |
| Config Getter | View→Presenter | `get_entered_primary()` | View extracts UI input |
| Config Setter | Presenter→Storage | `nv_storage_save()` | Presenter persists config |

### 11.4 Downstream Data Flow Implementation Details

The Model layer serves as the bridge between the energy meter firmware and the TouchGFX presentation layer. During each frame cycle, `Model::tick()` reads current measurement values from the shared `EM_PARAMS` structure, packages them into screen-specific data structures, and broadcasts them through the ModelListener interface.

### 11.5 Upstream Data Flow for Configuration

Configuration changes initiated through the user interface flow upstream from the View through the Presenter to the Model layer, which interfaces with the non-volatile storage subsystem.

When the user modifies a setting (such as PT ratio or communication parameters), the View captures the input and invokes a Presenter method:

```cpp
// SettingsView.cpp - User input capture
void Settings_PT_RatioView::handle_center_click() {
    // User confirmed new PT ratio value
    uint32_t new_primary = entered_primary_value;
    uint32_t new_secondary = entered_secondary_value;
    
    // Forward to Presenter for processing
    presenter->save_pt_ratio(new_primary, new_secondary);
}
```

The Presenter implements the save operation, potentially involving validation and non-volatile storage. After saving, it triggers navigation via the View's custom action:

```cpp
// Settings_PT_RatioPresenter.cpp - Configuration persistence
void Settings_PT_RatioPresenter::save_pt_ratio(uint32_t primary, uint32_t secondary) {
    // Update the firmware configuration structure
    EM_CONFIG.PT_Primary = primary;
    EM_CONFIG.PT_Secondary = secondary;
    
    // Persist to non-volatile storage
    nv_storage_save(&EM_CONFIG);
    
    // Navigate to success confirmation screen via View's custom action
    getView().takeToSaveSuccess();  // Action triggers "Change screen" interaction
}
```

### 11.6 Data Throttling Strategy

The display system operates at approximately 60 frames per second, but measurement data typically changes at much lower rates (once or twice per second). Updating display widgets every frame would consume unnecessary CPU cycles and could cause visible flickering if values change during mid-frame rendering.

The TorTitan implementation addresses this through frame-count-based throttling in the Model layer. A counter increments each frame, and data broadcasts occur only when the counter exceeds a threshold value. The threshold of 30 frames corresponds to approximately 500 milliseconds between updates, which provides responsive display while avoiding excessive update overhead.

#### 11.6.1 Throttling Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           DATA THROTTLING MECHANISM                                  │
│                      File: TouchGFX/gui/src/model/Model.cpp                         │
└─────────────────────────────────────────────────────────────────────────────────────┘

   TouchGFX Framework calls Model::tick() every frame (~16.67ms at 60 FPS)
                                    │
                                    ▼
         ┌──────────────────────────────────────────────────────────────┐
         │                    Model::tick()                             │
         │  ┌────────────────────────────────────────────────────────┐  │
         │  │  ticks++;           // Total frame counter             │  │
         │  │  data_update++;     // Throttle counter (resets)       │  │
         │  └────────────────────────────────────────────────────────┘  │
         └──────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
         ┌──────────────────────────────────────────────────────────────┐
         │              DUAL-CONDITION THROTTLE CHECK                   │
         │                                                              │
         │   if ((xTaskNotifyWait(...) == pdTRUE)    ◄── FreeRTOS       │
         │       && (data_update >= 30))             ◄── Frame count    │
         │                                               threshold      │
         │                                                              │
         │   Condition 1: Firmware signals new data ready               │
         │   Condition 2: At least 30 frames (~500ms) elapsed           │
         └──────────────────────────────────────────────────────────────┘
                │                                            │
        ┌───────┴───────┐                            ┌───────┴───────┐
        │  BOTH TRUE    │                            │ EITHER FALSE  │
        └───────┬───────┘                            └───────┬───────┘
                │                                            │
                ▼                                            ▼
  ┌──────────────────────────────┐              ┌──────────────────────────┐
  │  data_update = 0;  // Reset  │              │  Skip data broadcast     │
  │                              │              │  (wait for next frame)   │
  │  // Read firmware data       │              └──────────────────────────┘
  │  vaf_screen_data.voltage_ltn │
  │      = EM_PARAMS.Vpn;        │
  │  vaf_screen_data.current     │
  │      = EM_PARAMS.Iavg;       │
  │  ... (all parameters)        │
  │                              │
  │  // Broadcast to Presenter   │
  │  modelListener->set_vaf_data │
  │      (vaf_screen_data);      │
  └──────────────────────────────┘
```

#### 11.6.2 Implementation Code Example

```cpp
// File: TouchGFX/gui/src/model/Model.cpp

// Static counters declared at file scope
static uint8_t count = 0;

// In Model class initialization (Model.cpp constructor)
Model::Model() : modelListener(0) {
    ticks = 0;
    counts = 0;
    data_update = 0;      // Initialize throttle counter
    big_three_counts = 0;
}

void Model::tick() {
    // Called every frame by TouchGFX (~60 times per second)
    
    // Increment counters
    ticks++;
    data_update++;
    
    // Dual-condition check:
    // 1. xTaskNotifyWait: Non-blocking check if firmware signals new data
    // 2. data_update >= 30: Minimum 30 frames (~500ms) since last update
    uint32_t notification_value;
    if ((xTaskNotifyWait(0, 0xFFFFFFFF, &notification_value, 0) == pdTRUE)
            && (data_update >= 30)) {
        
        // Reset throttle counter
        data_update = 0;
        
        // Read current measurement data from firmware
        vaf_screen_data.voltage_ltn = EM_PARAMS.Vpn;
        vaf_screen_data.voltage_ltl = EM_PARAMS.Vpp;
        vaf_screen_data.current = EM_PARAMS.Iavg;
        vaf_screen_data.power_factor = EM_PARAMS.PFsys;
        vaf_screen_data.frequency = EM_PARAMS.Freq;
        
        // Broadcast to active screen's Presenter
        if (modelListener != 0 && current_screen == VAF_SCREEN) {
            modelListener->set_vaf_data(vaf_screen_data);
        }
        
        // Similar pattern for other screens...
        if (modelListener != 0 && current_screen == VR_DISTORTION_SCREEN) {
            modelListener->set_vthdr_graph_data(EM_PARAMS.HVR);
        }
        // ... additional screen-specific updates
    }
}
```

#### 11.6.3 Timing Calculations

| Parameter | Value | Calculation |
|-----------|-------|-------------|
| Display refresh rate | 60 FPS | TouchGFX framework default |
| Frame period | ~16.67 ms | 1000ms / 60 frames |
| Throttle threshold | 30 frames | Configured in code |
| Minimum update interval | ~500 ms | 30 × 16.67ms |
| Maximum updates per second | 2 | 1000ms / 500ms |
| CPU savings | ~93% | Only 2 of ~60 frames process data |

#### 11.6.4 Why Dual-Condition Throttling?

The TorTitan implementation uses a **dual-condition** throttle check combining FreeRTOS notifications with frame counting:

1. **FreeRTOS Notification (`xTaskNotifyWait`):**
   - The energy meter measurement task (in `Core/Src/EM.c`) signals when new measurement data is ready
   - Non-blocking check (timeout = 0) ensures tick() doesn't stall
   - Synchronizes display with actual data availability

2. **Frame Count Threshold (`data_update >= 30`):**
   - Prevents excessive updates even if firmware signals frequently
   - Ensures human-perceptible update rate (~500ms)
   - Provides consistent timing regardless of firmware timing variations

This dual approach provides **both responsiveness and efficiency**: updates happen promptly when new data arrives, but never more frequently than necessary for user perception.

#### 11.6.5 Screen-Specific Throttling

The throttle logic broadcasts data only to the **currently active screen**, avoiding wasted processing:

```cpp
// Only the active screen receives data
if (modelListener != 0 && current_screen == VAF_SCREEN) {
    modelListener->set_vaf_data(vaf_screen_data);
}
if (modelListener != 0 && current_screen == VR_DISTORTION_SCREEN) {
    modelListener->set_vthdr_graph_data(EM_PARAMS.HVR);
}
if (modelListener != 0 && current_screen == ALARM_HISTORY_SCREEN) {
    modelListener->populate_alarm_history();
}
// ... etc.
```

**Benefits:**
- No data processing for inactive screens
- Reduces memory bandwidth and CPU cycles
- Simplifies Presenter implementation (always receives relevant data)

---

## 12. Presenter-Model-View Communication Architecture

### 12.1 MVP Pattern Fundamentals

The Model-View-Presenter (MVP) architectural pattern provides the foundational structure for TouchGFX applications. This pattern establishes clear separation between data management (Model), visual presentation (View), and application logic (Presenter), enabling independent development and testing of each layer.

The MVP pattern in TouchGFX differs from traditional implementations in that the Model layer serves a dual role: it maintains application state and also provides the communication pathway between the energy meter firmware and the presentation layer. This dual role is appropriate for embedded applications where the "model" data originates from external hardware rather than user actions.

### 12.2 Complete MVP Architecture Diagram

The following diagram details all four MVP components, their files, responsibilities, and interactions:

```
╔═══════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
║                                    MVP ARCHITECTURE OVERVIEW                                                   ║
╠═══════════════════════════════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                                                ║
║    ┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐    ║
║    │                                         FIRMWARE LAYER                                               │    ║
║    │    Files: Core/Src/EM.c, Core/Src/Joystick.c, Core/Src/alarm_manager.c                              │    ║
║    │    Data:  EM_PARAMS (measurements), joystick_queue (input), trigger_alarm (alerts)                  │    ║
║    └─────────────────────────────────────────────┬───────────────────────────────────────────────────────┘    ║
║                                                  │ extern access                                               ║
║                                                  ▼                                                             ║
║    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    ║
║    ┃                                           MODEL                                                      ┃    ║
║    ┃━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┃    ║
║    ┃  Files:                                                                                              ┃    ║
║    ┃    - TouchGFX/gui/include/gui/model/Model.hpp                                                        ┃    ║
║    ┃    - TouchGFX/gui/src/model/Model.cpp                                                                ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃  RESPONSIBILITIES:                                                                                   ┃    ║
║    ┃    [1] Poll firmware data every frame (tick() called at 60 fps)                                     ┃    ║
║    ┃    [2] Maintain global application state (current_screen, auto_scroll_enabled, etc.)                ┃    ║
║    ┃    [3] Throttle data updates (broadcast every 30 frames = ~500ms)                                   ┃    ║
║    ┃    [4] Read joystick queue and dispatch navigation events                                           ┃    ║
║    ┃    [5] Monitor alarm flags and trigger alarm screen                                                 ┃    ║
║    ┃    [6] Broadcast data/events to active Presenter via ModelListener pointer                          ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃  KEY CODE:                                                                                           ┃    ║
║    ┃    class Model {                                                                                     ┃    ║
║    ┃        ModelListener* modelListener;  // Points to active Presenter                                 ┃    ║
║    ┃        eScreen current_screen;                                                                       ┃    ║
║    ┃        uint32_t data_update_counter;                                                                 ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        void tick() {                                                                                 ┃    ║
║    ┃            // Poll joystick                                                                          ┃    ║
║    ┃            if (xQueueReceive(joystick_queue, &event, 0)) {                                           ┃    ║
║    ┃                modelListener->handle_left_click();  // etc.                                          ┃    ║
║    ┃            }                                                                                         ┃    ║
║    ┃            // Throttled data broadcast                                                               ┃    ║
║    ┃            if (++data_update_counter >= 30) {                                                        ┃    ║
║    ┃                modelListener->set_vaf_data(vaf_data);                                                ┃    ║
║    ┃                data_update_counter = 0;                                                              ┃    ║
║    ┃            }                                                                                         ┃    ║
║    ┃        }                                                                                             ┃    ║
║    ┃    };                                                                                                ┃    ║
║    ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    ║
║                                       │ modelListener-> (calls interface methods)                             ║
║                                       ▼                                                                        ║
║    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    ║
║    ┃                                      MODELLISTENER (Interface)                                       ┃    ║
║    ┃━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┃    ║
║    ┃  File: TouchGFX/gui/include/gui/model/ModelListener.hpp                                              ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃  RESPONSIBILITIES:                                                                                   ┃    ║
║    ┃    [1] Define abstract interface for all events Model can broadcast                                 ┃    ║
║    ┃    [2] Provide empty default implementations (Presenters override only needed methods)              ┃    ║
║    ┃    [3] Decouple Model from specific Presenter implementations                                       ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃  INTERFACE DEFINITION:                                                                               ┃    ║
║    ┃    class ModelListener {                                                                             ┃    ║
║    ┃    public:                                                                                           ┃    ║
║    ┃        virtual ~ModelListener() {}                                                                   ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        // Navigation events (joystick)                                                               ┃    ║
║    ┃        virtual void handle_left_click() {}                                                           ┃    ║
║    ┃        virtual void handle_right_click() {}                                                          ┃    ║
║    ┃        virtual void handle_center_click() {}                                                         ┃    ║
║    ┃        virtual void handle_up_click() {}                                                             ┃    ║
║    ┃        virtual void handle_down_click() {}                                                           ┃    ║
║    ┃        virtual void handle_long_press() {}                                                           ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        // Data update events (one per screen type)                                                   ┃    ║
║    ┃        virtual void set_vaf_data(vaf_data_t data) {}                                                 ┃    ║
║    ┃        virtual void set_voltage_ln_data(voltage_data_t data) {}                                      ┃    ║
║    ┃        virtual void set_current_data(current_data_t data) {}                                         ┃    ║
║    ┃        virtual void set_phase_data(phase_data_t data) {}                                             ┃    ║
║    ┃        // ... more data setters for each screen                                                      ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        // System events                                                                              ┃    ║
║    ┃        virtual void trigger_alarm_screen() {}                                                        ┃    ║
║    ┃        virtual void back_to_VAF() {}                                                                 ┃    ║
║    ┃    };                                                                                                ┃    ║
║    ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    ║
║                                       │ Presenters implement this interface                                    ║
║                                       ▼                                                                        ║
║    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    ║
║    ┃                                         PRESENTER                                                    ┃    ║
║    ┃━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┃    ║
║    ┃  Files (per screen):                                                                                 ┃    ║
║    ┃    - TouchGFX/gui/include/gui/vaf_screen/VAFPresenter.hpp                                            ┃    ║
║    ┃    - TouchGFX/gui/src/vaf_screen/VAFPresenter.cpp                                                    ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃  RESPONSIBILITIES:                                                                                   ┃    ║
║    ┃    [1] Implement ModelListener interface (override relevant methods only)                           ┃    ║
║    ┃    [2] Forward data from Model to View via getView().set_xxx()                                      ┃    ║
║    ┃    [3] Handle screen lifecycle (activate/deactivate)                                                ┃    ║
║    ┃    [4] Provide data access methods for View (getter methods)                                        ┃    ║
║    ┃    [5] Process configuration saves from View                                                        ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃  KEY CODE:                                                                                           ┃    ║
║    ┃    class VAFPresenter : public touchgfx::Presenter, public ModelListener {                           ┃    ║
║    ┃        VAFView& view;  // Reference to associated View                                              ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        void activate() override {                                                                    ┃    ║
║    ┃            // Called when screen becomes active                                                      ┃    ║
║    ┃        }                                                                                             ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        void deactivate() override {                                                                  ┃    ║
║    ┃            // Called when leaving screen                                                             ┃    ║
║    ┃        }                                                                                             ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        // ModelListener override - receives data from Model                                         ┃    ║
║    ┃        void set_vaf_data(vaf_data_t data) override {                                                 ┃    ║
║    ┃            getView().set_vaf_data(data);  // Forward to View                                        ┃    ║
║    ┃        }                                                                                             ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        // Getter for View to access Model data                                                       ┃    ║
║    ┃        bool is_autoscroll_on() { return model->getAutoScrollEnabled(); }                             ┃    ║
║    ┃    };                                                                                                ┃    ║
║    ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    ║
║                                       │ getView().set_xxx() / presenter->get_xxx()                            ║
║                                       ▼                                                                        ║
║    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    ║
║    ┃                                           VIEW                                                       ┃    ║
║    ┃━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┃    ║
║    ┃  Files (per screen):                                                                                 ┃    ║
║    ┃    - TouchGFX/gui/include/gui/vaf_screen/VAFView.hpp                                                 ┃    ║
║    ┃    - TouchGFX/gui/src/vaf_screen/VAFView.cpp                                                         ┃    ║
║    ┃    - TouchGFX/generated/gui_generated/.../VAFViewBase.hpp (DO NOT EDIT - auto-generated)            ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃  RESPONSIBILITIES:                                                                                   ┃    ║
║    ┃    [1] Contain and manage widget hierarchy (TextAreas, Graphs, Images, Containers)                  ┃    ║
║    ┃    [2] Implement set_xxx() methods to receive and display data                                      ┃    ║
║    ┃    [3] Format values for display (scaling, precision, units)                                        ┃    ║
║    ┃    [4] Handle widget invalidation for efficient rendering                                           ┃    ║
║    ┃    [5] Implement handle_xxx_click() for navigation (calls custom actions)                           ┃    ║
║    ┃    [6] Manage screen setup and teardown                                                              ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃  KEY CODE:                                                                                           ┃    ║
║    ┃    class VAFView : public VAFViewBase {                                                              ┃    ║
║    ┃        VAFPresenter* presenter;  // Access to Presenter                                             ┃    ║
║    ┃        Unicode::UnicodeChar voltage_buffer[20];                                                      ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        void setupScreen() override {                                                                 ┃    ║
║    ┃            VAFViewBase::setupScreen();                                                               ┃    ║
║    ┃            presenter->set_current_screen(VAF_SCREEN);                                                ┃    ║
║    ┃        }                                                                                             ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        void set_vaf_data(vaf_data_t data) {                                                          ┃    ║
║    ┃            // Format and display                                                                     ┃    ║
║    ┃            snprintf(temp, sizeof(temp), "%.1f", data.voltage_ltn);                                   ┃    ║
║    ┃            Unicode::fromUTF8((uint8_t*)temp, voltage_buffer, 20);                                    ┃    ║
║    ┃            voltage_text.invalidate();                                                                ┃    ║
║    ┃        }                                                                                             ┃    ║
║    ┃                                                                                                      ┃    ║
║    ┃        void handle_right_click() {                                                                   ┃    ║
║    ┃            takeToPowerMaxDemand();  // Custom action triggers screen change                         ┃    ║
║    ┃        }                                                                                             ┃    ║
║    ┃    };                                                                                                ┃    ║
║    ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    ║
║                                       │ widget.invalidate()                                                    ║
║                                       ▼                                                                        ║
║    ┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐    ║
║    │                                       WIDGETS & RENDERING                                            │    ║
║    │    TouchGFX Widgets: TextArea, Graph, Box, Image, Container, Circle, Line, Shape                   │    ║
║    │    Rendering: DMA2D hardware acceleration writes to LCD framebuffer via FMC                        │    ║
║    └─────────────────────────────────────────────────────────────────────────────────────────────────────┘    ║
║                                                                                                                ║
╚═══════════════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

### 12.3 Component Interaction Summary

| Component | File Pattern | Creates | Receives From | Sends To |
|-----------|--------------|---------|---------------|----------|
| **Model** | `gui/src/model/Model.cpp` | Data structures, events | Firmware (extern) | ModelListener (Presenter) |
| **ModelListener** | `gui/include/gui/model/ModelListener.hpp` | Interface definition | - | - |
| **Presenter** | `gui/src/<screen>/<Screen>Presenter.cpp` | Formatted calls | Model (via interface) | View (via getView()) |
| **View** | `gui/src/<screen>/<Screen>View.cpp` | Widget updates | Presenter (set methods) | Widgets (invalidate) |

### 12.4 Component Responsibilities Summary

Each layer in the MVP architecture has well-defined responsibilities:

**Model Layer:** The Model maintains global application state including the current screen identifier, alarm status flags, and auto-scroll configuration. It reads measurement data from the firmware's shared data structures during each frame cycle and broadcasts updates through the ModelListener interface. The Model is instantiated once and persists throughout the application lifetime.

**ModelListener Interface:** This abstract interface defines all callback methods that Presenters may implement. It includes data update methods (one per screen type), navigation event methods (joystick actions), and system event methods (alarm triggers). The interface uses empty default implementations, allowing Presenters to override only the methods relevant to their screen.

**Presenter Layer:** Each screen has a dedicated Presenter that implements the ModelListener interface. The Presenter handles navigation decisions, data transformation if needed, and communication with its associated View. Only the Presenter for the currently active screen receives and processes events; other Presenters exist but remain dormant.

**View Layer:** Views contain the widget hierarchy and handle visual presentation. They receive formatted data from their Presenter and update widget content accordingly. Views should not contain business logic or navigation decisions; these responsibilities belong to the Presenter.

### 12.5 File Organization

The MVP components reside in specific locations within the project structure:

| Component | Header Location | Source Location | Responsibilities |
|-----------|-----------------|-----------------|------------------|
| Model | `gui/include/gui/model/Model.hpp` | `gui/src/model/Model.cpp` | State management, firmware interface |
| ModelListener | `gui/include/gui/model/ModelListener.hpp` | (interface only) | Event callback declarations |
| Presenter | `gui/include/gui/<screen>/Presenter.hpp` | `gui/src/<screen>/Presenter.cpp` | Screen logic, navigation |
| View | `gui/include/gui/<screen>/View.hpp` | `gui/src/<screen>/View.cpp` | Widget management, display |

### 12.6 Presenter Lifecycle

Presenters follow a defined lifecycle managed by the TouchGFX framework. Understanding this lifecycle is essential for proper resource management and data handling.

**Activation:** When a screen becomes active (either at startup or through navigation), the framework calls the Presenter's `activate()` method. This method should perform any initialization required for the screen, such as requesting initial data from the Model or configuring screen-specific state.

**Active Operation:** While active, the Presenter receives callbacks through the ModelListener interface. It processes these callbacks, potentially transforming data before forwarding to the View or making navigation decisions based on user input.

**Deactivation:** When navigation moves to a different screen, the framework calls `deactivate()` on the outgoing Presenter. This method should release any resources and stop any operations that should not continue while the screen is inactive.

```cpp
// Example Presenter lifecycle implementation
class VAFPresenter : public touchgfx::Presenter, public ModelListener {
public:
    virtual void activate() override {
        // Screen becoming active - request initial data
        // Set up any screen-specific state
    }
    
    virtual void deactivate() override {
        // Screen becoming inactive - clean up
    }
    
    // ModelListener overrides for data
    virtual void set_vaf_data(vaf_data_t data) override {
        // Forward data to View
        getView().set_vaf_data(data);
    }
    
    // Note: In this project, handle_xxx_click() methods are implemented
    // in the View class, not the Presenter. See Section 5.3 for examples.
};
```

### 12.7 Key Design Rules

Several design rules ensure proper MVP implementation:

**View Independence from Model:** Views must never access the Model directly. All data and events flow through the Presenter, maintaining separation of concerns and enabling the View to be tested independently.

**Broadcast to All, Process in One:** The Model broadcasts events to the ModelListener interface without knowing which Presenter is active. The currently active Presenter processes the event; inactive Presenters have empty default implementations that ignore the broadcast.

**Single Active Presenter:** Only one Presenter is active at any time, corresponding to the currently displayed screen. This simplifies event handling and ensures that input events are processed by the appropriate screen logic.

**Navigation Ownership:** Navigation decisions belong in the Presenter layer. Views should not directly invoke screen transitions; instead, they inform the Presenter of user actions, and the Presenter decides whether and where to navigate.

### 12.8 Complete Worked Example: VAF Screen Architecture

The VAF (Voltage-Amperage-Frequency) screen serves as the primary display screen for the energy meter and provides an excellent example of the complete MVP implementation. This section walks through the actual code structure and data flow.

#### 12.8.1 VAF Screen Component Overview

```
VAF Screen Files:                    Data Structure (vaf_screen_data_t):
  - VAFView.cpp                        - voltage_ltn
  - VAFPresenter.cpp                   - voltage_ltl  
  - Generated Base Classes             - current
                                       - power_factor
        ↓                              - frequency
  Data → View → Widgets (VLN/VLL/Current/PF/Freq Cards)
```

#### 12.8.2 VAFView.cpp Key Implementation Patterns

The VAFView demonstrates several important patterns from the actual codebase:

**Initialization and Screen Setup:**
```cpp
// From VAFView.cpp - Screen initialization
void VAFView::setupScreen() {
    VAFViewBase::setupScreen();                    // Call generated base setup
    presenter->set_current_screen(VAF_SCREEN);     // Inform Model of active screen
    selected_index = presenter->get_vaf_select_index();  // Restore selection state
    
    // Configure card selection based on saved index
    switch (selected_index) {
        case 0: selectIndex0(); break;
        case 1: selectIndex1(); break;
        // ... etc
    }
    
    // Retrieve and display connectivity status
    bool wifi, ble;
    presenter->get_ble_wifi_status(&ble, &wifi);
    set_ble_wifi(ble, wifi);
}
```

**Data Formatting with formatWithDigits:**
```cpp
// From VAFView.cpp - Smart digit formatting
void VAFView::formatWithDigits(float val, int totalDigits,
        Unicode::UnicodeChar *buffer, size_t bufferSize) {
    
    double absVal = fabs(val);
    int intDigits = (absVal < 1.0) ? 1 : (int) floor(log10(absVal)) + 1;
    if (val < 0.0) ++intDigits;  // Account for minus sign
    
    int decimalDigits = totalDigits - intDigits;
    if (decimalDigits < 0) decimalDigits = 0;
    if (decimalDigits > 9) decimalDigits = 9;
    
    // Truncate (not round) to prevent overstating values
    double pow10 = pow(10.0, decimalDigits);
    double truncatedVal = trunc(val * pow10) / pow10;
    
    char formatStr[8];
    snprintf(formatStr, sizeof(formatStr), "%%.%df", decimalDigits);
    
    char temp[50];
    snprintf(temp, sizeof(temp), formatStr, truncatedVal);
    Unicode::fromUTF8((uint8_t*) temp, buffer, bufferSize);
}
```

**Navigation with Custom Actions:**
```cpp
// From VAFView.cpp - Navigation handlers
void VAFView::handle_left_click() {
    takeToAlarmHistory();  // Custom action defined in Designer
}

void VAFView::handle_right_click() {
    takeToPowerMaxDemand();  // Custom action defined in Designer
}

// Auto-scroll implementation
void VAFView::handleTickEvent() {
    if (++ticks >= 300 && presenter->is_autoscroll_on()) {
        ticks = 0;
        takeToPowerMaxDemand();  // Auto-navigate after ~5 seconds
    }
}
```

#### 12.8.3 VAF Screen Navigation Context

```
Alarm History ←─(L/R)─→ VAF Home ←─(L/R)─→ Power Max Demand
                           │
                     (Center) → Settings Menu
                     (Auto-scroll) → Power Max Demand
```

---

## 13. Dynamic Text Update Handling

### 13.1 Complete Update Implementation

Dynamic text updates represent one of the most common operations in the TorTitan interface, as measurement values continuously change and must be reflected on the display. The update process involves several stages: data validation, unit scaling, value formatting, Unicode conversion, and widget invalidation. This section documents the complete implementation pattern used throughout the project.

```
Raw Value → Valid? ──No──→ "--" ─────────────────────────┐
              │Yes                                        │
              ↓                                           │
         Auto-scale (GV/MV/KV/V)                          │
              ↓                                           │
         trunc (not round)                                │
              ↓                                           │
           snprintf                                       │
              ↓                                           │
          fromUTF8                                        │
              ↓                                           │
          invalidate() ←──────────────────────────────────┘
              ↓
          DMA2D Render
```

The following comprehensive example demonstrates the full update sequence for voltage display:

```cpp
void Voltage_LNView::set_voltage_data(voltage_data_t data) {
    float r = data.vr;
    
    // Stage 1: Validate data integrity
    // NaN and infinity values indicate measurement errors
    if (isnan(r) || isinf(r)) {
        // Display placeholder for invalid data
        Unicode::fromUTF8((uint8_t*)"--", vr_valueBuffer, VR_SIZE);
        Unicode::fromUTF8((uint8_t*)"", vr_unitBuffer, UNIT_SIZE);
    } else {
        // Stage 2: Apply automatic unit scaling
        // Scale values to appropriate engineering units for readability
        char vr_unit[4] = "V";
        if (fabs(r) >= 1e9) { 
            r = r / 1e9; 
            strcpy(vr_unit, "GV"); 
        } else if (fabs(r) >= 1e6) { 
            r = r / 1e6; 
            strcpy(vr_unit, "MV"); 
        } else if (fabs(r) >= 1e3) { 
            r = r / 1e3; 
            strcpy(vr_unit, "KV"); 
        }
        
        // Stage 3: Apply truncation (not rounding)
        // Truncation ensures displayed values never exceed actual measurements
        double truncated = trunc(r * 100.0) / 100.0;
        
        // Stage 4: Format as string
        char temp[50];
        snprintf(temp, sizeof(temp), "%.2f", truncated);
        
        // Stage 5: Convert to Unicode
        Unicode::fromUTF8((uint8_t*)temp, vr_valueBuffer, VR_SIZE);
        Unicode::fromUTF8((uint8_t*)vr_unit, vr_unitBuffer, UNIT_SIZE);
    }
    
    // Stage 6: Invalidate only affected widgets
    // Targeted invalidation optimizes rendering performance
    vr_text.invalidate();
    vr_unit_text.invalidate();
}
```

### 13.2 Truncation vs. Rounding Decision

The TorTitan project deliberately uses truncation rather than rounding for displayed measurement values. This design decision reflects the principle that displayed values should never exceed the actual measured quantity. In measurement applications, overstating values (which rounding can cause) is generally considered more problematic than understating them.

The truncation implementation uses the mathematical `trunc()` function combined with scaling:

```cpp
// Truncate to 2 decimal places
// Example: 230.456 → 230.45 (not 230.46 as rounding would produce)
double truncated = trunc(value * 100.0) / 100.0;
```

The scaling factor (100.0 for two decimal places) should match the format string precision (%.2f) to ensure consistent display. For different precision requirements, adjust both values correspondingly: use 10.0 and %.1f for one decimal place, or 1000.0 and %.3f for three decimal places.

### 13.3 Performance Optimization for Text Updates

Text updates can become a performance bottleneck if not implemented carefully. The following practices ensure optimal performance:

**Targeted Invalidation:** Always invalidate specific widgets rather than the entire screen. The code `vr_text.invalidate()` marks only that widget for redrawing, while `invalidate()` (without a target) would mark the entire screen. Targeted invalidation dramatically reduces rendering workload.

**Update Throttling:** Data updates should not occur every frame. The Model layer implements throttling that limits updates to approximately 2 Hz (every 30 frames at 60 fps). This rate is sufficient for human perception while minimizing CPU usage.

**Pre-computation:** When complex formatting or calculations are required, perform them in the Presenter or Model layer rather than the View. This separates computational load from rendering and allows potential optimization or caching.

**Conditional Updates:** When possible, compare new values against previously displayed values and skip updates if unchanged. This optimization is particularly valuable for slowly-changing measurements.

### 13.4 Handling Multiple Value Updates

When updating multiple related values (such as R, Y, and B phase voltages), group the updates logically but maintain individual widget invalidation:

```cpp
void ThreePhaseView::set_phase_data(phase_data_t data) {
    // Update all phase value buffers
    formatPhaseValue(data.r, r_buffer, R_BUFFER_SIZE, r_unit);
    formatPhaseValue(data.y, y_buffer, Y_BUFFER_SIZE, y_unit);
    formatPhaseValue(data.b, b_buffer, B_BUFFER_SIZE, b_unit);
    
    // Invalidate all affected widgets
    // TouchGFX batches invalidations within a frame
    r_text.invalidate();
    r_unit_text.invalidate();
    y_text.invalidate();
    y_unit_text.invalidate();
    b_text.invalidate();
    b_unit_text.invalidate();
}
```

The TouchGFX framework efficiently handles multiple invalidations within a single frame, combining overlapping regions and optimizing the rendering pass.

---

## 14. VLN/VLL Waveform Graph Implementation

### 14.1 Graph Architecture Overview

The voltage and current waveform displays utilize the Static_Graph custom container, which encapsulates three TouchGFX Graph widgets for simultaneous display of R, Y, and B phase waveforms. This container provides a reusable visualization component that appears on multiple screens including Line-to-Neutral voltage (VLN), Line-to-Line voltage (VLL), and current measurement displays.

The Graph widget class (`touchgfx::Graph<128>`) provides the underlying rendering capability, supporting up to 128 data points with various line styles and colors. The TorTitan implementation uses 64 data points per phase, representing one complete electrical cycle from 0° to 360°.

### 14.2 Graph Configuration Parameters

The waveform display configuration establishes the visual characteristics of the graph:

| Parameter | Value | Description |
|-----------|-------|-------------|
| Data Points | 64 per phase | One complete cycle representation |
| X-Axis Range | 0° to 360° | Angular position within cycle |
| Y-Axis Range | Dynamic | Based on PT ratio configuration |
| Line Width | 3-4 pixels | Visible at display resolution |
| R-Phase Color | RGB(248, 40, 90) | Red spectrum |
| Y-Phase Color | RGB(246, 192, 0) | Yellow spectrum |
| B-Phase Color | RGB(14, 186, 246) | Cyan/Blue spectrum |

### 14.3 Data Source Integration

The waveform data originates from the energy meter firmware, which captures 256 samples per cycle through the ADC subsystem. The firmware stores these samples in global arrays that are accessible to the TouchGFX layer through extern declarations:

```cpp
// External firmware arrays declared in Model.cpp
extern float Vr[256], Vy[256], Vb[256];       // Line-to-Neutral waveforms
extern float Vry[256], Vyb[256], Vbr[256];    // Line-to-Line waveforms
extern float Ir[256], Iy[256], Ib[256];       // Current waveforms
```

The 256-sample arrays provide higher resolution than displayed. The graph container subsamples this data, using every fourth sample to produce 64 display points. This subsampling approach allows the firmware to maintain high-resolution data for calculation purposes while the display shows a visually appropriate representation.

### 14.4 Graph Rendering Implementation

The Static_Graph container implements the `set_graph_data()` method to receive waveform data and update the display:

```cpp
void Static_Graph::set_graph_data(float* r_data, float* y_data, float* b_data) {
    // Calculate X-axis step for 64 points across 360 degrees
    const float step = 360.0f / 63.0f;  // Approximately 5.71 degrees per point
    
    // Clear existing data points
    r_graph.clear();
    y_graph.clear();
    b_graph.clear();
    
    // Populate graphs with new data
    for (int i = 0; i < 64; i++) {
        float x = i * step;  // X coordinate in degrees
        
        // Add data points to each phase graph
        r_graph.addDataPoint(x, r_data[i * 4]);  // Subsample from 256 to 64
        y_graph.addDataPoint(x, y_data[i * 4]);
        b_graph.addDataPoint(x, b_data[i * 4]);
    }
    
    // Mark container for redrawing
    invalidate();
}
```

### 14.5 Dynamic Y-Axis Scaling

The Y-axis range must accommodate the configured Potential Transformer (PT) ratio, which scales the measured voltage to the actual system voltage. The Y-axis calculation differs between VLN and VLL displays:

```cpp
// Line-to-Neutral (VLN) Y-axis calculation
// Base value of 400V represents the maximum secondary voltage
float maxY_VLN = 400.0f * (EM_CONFIG.PT_Primary / EM_CONFIG.PT_Secondary);

// Line-to-Line (VLL) Y-axis calculation  
// Includes √3 multiplier for phase-to-phase voltage relationship
float maxY_VLL = 1.732f * 400.0f * (EM_CONFIG.PT_Primary / EM_CONFIG.PT_Secondary);
```

The graph range is set during container initialization or when the PT ratio configuration changes:

```cpp
void Static_Graph::setYAxisRange(float maxY) {
    r_graph.setGraphRange(0, 360, -maxY, maxY);
    y_graph.setGraphRange(0, 360, -maxY, maxY);
    b_graph.setGraphRange(0, 360, -maxY, maxY);
}
```

### 14.6 Waveform Rendering Mathematics

The waveform graph visualization involves several mathematical transformations to convert raw ADC samples into properly scaled and positioned pixel coordinates on the display.

#### 14.6.1 Sinusoidal Signal Representation

The voltage and current signals in an AC electrical system are fundamentally sinusoidal. A pure sinusoidal voltage waveform is described by:

$$v(t) = V_m \sin(\omega t + \phi)$$

Where:
- $V_m$ is the peak amplitude (volts)
- $\omega = 2\pi f$ is the angular frequency (rad/s)
- $f$ is the frequency (50 Hz or 60 Hz depending on grid)
- $\phi$ is the initial phase angle (radians)
- $t$ is time (seconds)

For a 50 Hz system, one complete cycle spans $T = 1/f = 20$ ms.

#### 14.6.2 Angular Domain Representation

Rather than plotting voltage against time, the TorTitan waveform display uses angular position as the x-axis, with one complete cycle represented from 0° to 360°. This representation is achieved by the substitution:

$$\theta = \omega t = 2\pi f t$$

Converting from time domain to angular domain:

$$v(\theta) = V_m \sin(\theta + \phi)$$

The angular representation has the advantage of being frequency-independent—the same display format works for both 50 Hz and 60 Hz systems.

#### 14.6.3 Sample-to-Angle Mapping

The firmware ADC captures 256 samples per electrical cycle, uniformly distributed over the cycle period. Each sample corresponds to an angular increment of:

$$\Delta\theta = \frac{360°}{256} = 1.406°$$

For display purposes, the graph shows 64 points (subsampling by factor of 4). The x-coordinate for each display point is:

$$x_i = i \times \frac{360°}{63} \quad \text{for } i \in [0, 63]$$

Note: 63 is used (not 64) because the 64 points span from 0° to 360° inclusive, creating 63 intervals.

```cpp
// X-axis mapping in Static_Graph::set_data()
const float step = 360.0f / 63.0f;  // ≈ 5.714° per point

for (int i = 0; i < 64; i++) {
    float x = i * step;  // Angular position: 0°, 5.71°, 11.43°, ..., 360°
    r_graph.addDataPoint(x, voltage[i]);
}
```

#### 14.6.4 Y-Axis Scaling with Potential Transformer Ratio

The energy meter measures secondary voltage from potential transformers (PTs) that step down the high system voltage. The display must scale values to show actual system voltage:

$$V_{system} = V_{measured} \times \frac{PT_{primary}}{PT_{secondary}}$$

**Example:**
- PT Ratio: 11000V:110V (100:1)
- Measured peak voltage: 155.6V (110V RMS × √2)
- Displayed voltage: 155.6 × 100 = 15,560V peak

The Y-axis range is set to accommodate the maximum expected voltage:

$$Y_{max} = 400V \times \frac{PT_{primary}}{PT_{secondary}}$$

The 400V base value represents approximately the peak of a 275V RMS secondary voltage with margin.

#### 14.6.5 Line-to-Line Voltage Relationship

For Line-to-Line (VLL) waveforms, an additional scaling factor accounts for the $\sqrt{3}$ relationship between line-to-neutral and line-to-line voltages in three-phase systems:

$$V_{LL} = \sqrt{3} \times V_{LN} \approx 1.732 \times V_{LN}$$

This relationship arises from the vector addition of two 120°-separated phase voltages:

$$\vec{V_{RY}} = \vec{V_R} - \vec{V_Y}$$

The magnitude of this difference is:
$$|V_{RY}| = |V_{LN}| \times \sqrt{1^2 + 1^2 + 2 \times 1 \times 1 \times \cos(120°)} = |V_{LN}| \times \sqrt{3}$$

#### 14.6.6 Graph Coordinate System

The TouchGFX Graph widget uses a coordinate system where:
- **X-axis:** Represents the data domain (0° to 360°)
- **Y-axis:** Represents the data range (-maxY to +maxY)
- **Origin:** Bottom-left corner of the graph area

The widget automatically maps data coordinates to pixel coordinates based on the configured range:

$$x_{pixel} = \frac{x_{data} - x_{min}}{x_{max} - x_{min}} \times W_{graph}$$

$$y_{pixel} = H_{graph} - \frac{y_{data} - y_{min}}{y_{max} - y_{min}} \times H_{graph}$$

Where $W_{graph}$ and $H_{graph}$ are the graph widget dimensions in pixels.

### 14.7 Graph Widget Configuration

The individual graph widgets require proper configuration for visual appearance. This configuration typically occurs in the container's initialization or the generated base class:

```cpp
// Configure graph appearance
graph.setGraphAreaMargin(10, 10, 10, 10);  // Margins around plot area
graph.setGraphRange(0, 360, -maxY, maxY);   // Axis ranges

// Add line element with visual styling
GraphElementLine& lineElement = graph.addGraphElement<GraphElementLine>();
lineElement.setColor(Color::getColorFromRGB(248, 40, 90));  // Phase color
lineElement.setLineWidth(3);  // Line thickness in pixels
```

---

## 15. Power Quality and Harmonic Distortion Graphs

### 15.1 Harmonic Analysis Display Overview

The TorTitan interface provides comprehensive harmonic analysis visualization through dedicated screens for voltage and current Total Harmonic Distortion (THD). Six distortion screens display individual harmonic magnitudes for each phase: VR, VY, VB for voltage harmonics, and IR, IY, IB for current harmonics. Additionally, summary screens (VTHD_Total_Distortion and ITHD_Total_Distortion) provide aggregate THD percentages.

Harmonic distortion analysis is critical for power quality assessment. Non-sinusoidal waveforms contain frequency components at integer multiples of the fundamental frequency (50 Hz or 60 Hz). These harmonic components cause various power system problems including increased losses, equipment heating, and interference with sensitive electronics.

### 15.2 Distortion_Graph Container Architecture

The Distortion_Graph custom container renders harmonic magnitude data as a horizontal bar chart. The container displays 16 harmonic orders, focusing on odd harmonics which are most significant in typical power systems:

```
Displayed Harmonic Orders: 1st (fundamental), 3rd, 5th, 7th, 9th, 11th, 13th, 15th,
                           17th, 19th, 21st, 23rd, 25th, 27th, 29th, 31st
```

Each harmonic is represented by a horizontal bar widget whose width is proportional to the harmonic magnitude as a percentage of the fundamental.

### 15.3 Bar Chart Rendering Implementation

The Distortion_Graph container receives harmonic data through its setter method and updates bar widths accordingly:

```cpp
void Distortion_Graph::set_distortion_data(float data[16]) {
    const int GRAPH_WIDTH = 274;  // Maximum bar width in pixels
    
    for (int i = 0; i < 16; i++) {
        // Calculate bar width proportional to harmonic magnitude
        // data[i] is the harmonic magnitude as percentage (0-100)
        int width = (int)((data[i] / 100.0f) * GRAPH_WIDTH);
        
        // Ensure minimum visibility for non-zero values
        if (data[i] > 0.0f && width < 1) {
            width = 1;
        }
        
        // Update bar widget
        bars[i].setWidth(width);
        bars[i].setColor(Color::getColorFromRGB(255, 0, 0));  // Red bars
        bars[i].invalidate();
    }
}
```

The bar widgets (bars[0] through bars[15]) are Box widgets defined in the container's Designer layout, positioned vertically with appropriate labels indicating the harmonic order.

### 15.4 Data Source for Harmonic Information

The energy meter firmware calculates harmonic magnitudes through FFT (Fast Fourier Transform) analysis of the sampled waveforms. These calculated values are stored in the EM_PARAMS structure:

```cpp
// Harmonic data arrays in EM_PARAMS structure
extern PARAMS_t EM_PARAMS;

// Voltage harmonics (percentage of fundamental)
// EM_PARAMS.HVR[16] - R-phase voltage harmonics
// EM_PARAMS.HVY[16] - Y-phase voltage harmonics  
// EM_PARAMS.HVB[16] - B-phase voltage harmonics

// Current harmonics (percentage of fundamental)
// EM_PARAMS.HIR[16] - R-phase current harmonics
// EM_PARAMS.HIY[16] - Y-phase current harmonics
// EM_PARAMS.HIB[16] - B-phase current harmonics

// Total Harmonic Distortion percentages
// EM_PARAMS.VTHD_R, VTHD_Y, VTHD_B - Voltage THD per phase
// EM_PARAMS.ITHD_R, ITHD_Y, ITHD_B - Current THD per phase
```

### 15.5 THD Summary Screen Implementation

The THD summary screens display aggregate distortion percentages for all three phases, allowing quick assessment of overall power quality. These screens use wildcard text areas to display the calculated THD values:

```cpp
void VTHD_Total_DistortionView::set_thd_data(thd_data_t data) {
    char temp[20];
    
    // Format R-phase THD
    snprintf(temp, sizeof(temp), "%.1f%%", data.vthd_r);
    Unicode::fromUTF8((uint8_t*)temp, vthd_r_buffer, BUFFER_SIZE);
    
    // Format Y-phase THD
    snprintf(temp, sizeof(temp), "%.1f%%", data.vthd_y);
    Unicode::fromUTF8((uint8_t*)temp, vthd_y_buffer, BUFFER_SIZE);
    
    // Format B-phase THD
    snprintf(temp, sizeof(temp), "%.1f%%", data.vthd_b);
    Unicode::fromUTF8((uint8_t*)temp, vthd_b_buffer, BUFFER_SIZE);
    
    // Invalidate all THD display widgets
    vthd_r_text.invalidate();
    vthd_y_text.invalidate();
    vthd_b_text.invalidate();
}
```

### 15.6 Harmonic Bar Graph Mathematical Foundation

The harmonic bar graph provides a visual representation of the frequency domain components present in the measured electrical signal. Understanding the underlying mathematics is essential for both interpreting the display and maintaining the rendering code.

#### 15.6.1 Fourier Series Decomposition

Any periodic waveform can be decomposed into a sum of sinusoidal components at integer multiples of the fundamental frequency. For a periodic function $f(t)$ with period $T$ and fundamental frequency $f_1 = 1/T$, the Fourier series representation is:

$$f(t) = A_0 + \sum_{n=1}^{\infty} [A_n \cos(n \omega_1 t) + B_n \sin(n \omega_1 t)]$$

Where:
- $A_0$ is the DC offset component
- $\omega_1 = 2\pi f_1$ is the fundamental angular frequency
- $n$ is the harmonic order (n=1 is fundamental, n=2 is second harmonic, etc.)
- $A_n$ and $B_n$ are the Fourier coefficients for the $n$-th harmonic

The magnitude of the $n$-th harmonic is computed as:

$$M_n = \sqrt{A_n^2 + B_n^2}$$

#### 15.6.2 Harmonic Percentage Calculation

In power quality analysis, harmonic magnitudes are conventionally expressed as a percentage of the fundamental component. This normalization allows comparison across different voltage and current levels:

$$H_n\% = \frac{M_n}{M_1} \times 100\%$$

Where $M_1$ is the magnitude of the fundamental (n=1) component. The Total Harmonic Distortion (THD) aggregates all harmonic content into a single metric:

$$THD\% = \frac{\sqrt{\sum_{n=2}^{N} M_n^2}}{M_1} \times 100\%$$

The TorTitan firmware performs this calculation using FFT analysis of 256 ADC samples per cycle, computing harmonics up to the 31st order.

#### 15.6.3 Horizontal Bar Width Calculation

The Distortion_Graph container renders each harmonic as a horizontal bar whose width is proportional to its magnitude percentage. The linear scaling formula maps the percentage value (0-100%) to pixel width (0 to container width):

$$W_{pixels} = \frac{H_n\%}{100} \times W_{container}$$

In the TorTitan implementation, the graph container width is defined by the `graph_container` Box widget. The actual code implementation:

```cpp
// Container width obtained from the graph_container Box widget
int containerWidth = graph_container.getWidth();  // Typically 274 pixels

// Linear scaling from percentage to pixels
int barWidth = static_cast<int>((harmonicPercent / 100.0f) * containerWidth);
```

**Worked Example:**
- Suppose the 3rd harmonic magnitude is 8.5% of fundamental
- Container width = 274 pixels
- Bar width = (8.5 / 100) × 274 = 23.29 pixels ≈ 23 pixels

#### 15.6.4 Why Odd Harmonics Are Displayed

The TorTitan harmonic display focuses on odd harmonics (1st, 3rd, 5th, 7th, ..., 31st) because these are the dominant harmonic components in typical three-phase power systems. Even harmonics are generally absent or minimal due to the symmetrical nature of most nonlinear loads.

Mathematically, a waveform with half-wave symmetry (where $f(t + T/2) = -f(t)$) contains only odd harmonics. This property applies to most power electronic equipment:
- Variable frequency drives
- Switched-mode power supplies
- LED lighting drivers
- Uninterruptible power supplies (UPS)

The fundamental frequency and odd harmonics (3rd, 5th, 7th, etc.) carry nearly all the harmonic energy in these loads.

#### 15.6.5 Harmonic Order to Array Index Mapping

The display shows 16 bars representing harmonics at orders 1, 3, 5, 7, 9, 11, 13, 15, 17, 19, 21, 23, 25, 27, 29, and 31. The mapping between array index and harmonic order follows the pattern:

$$\text{Harmonic Order} = 2i + 1 \quad \text{where } i \in [0, 15]$$

| Array Index | Harmonic Order | Frequency (50 Hz fundamental) |
|-------------|----------------|-------------------------------|
| 0 | 1st (Fundamental) | 50 Hz |
| 1 | 3rd | 150 Hz |
| 2 | 5th | 250 Hz |
| 3 | 7th | 350 Hz |
| 4 | 9th | 450 Hz |
| 5 | 11th | 550 Hz |
| 6 | 13th | 650 Hz |
| 7 | 15th | 750 Hz |
| 8 | 17th | 850 Hz |
| 9 | 19th | 950 Hz |
| 10 | 21st | 1050 Hz |
| 11 | 23rd | 1150 Hz |
| 12 | 25th | 1250 Hz |
| 13 | 27th | 1350 Hz |
| 14 | 29th | 1450 Hz |
| 15 | 31st | 1550 Hz |

#### 15.6.6 Visual Scaling Considerations

The linear scaling approach used in the implementation has important implications for visual interpretation:

**Linearity:** A harmonic at 20% appears as a bar twice as long as one at 10%. This linear relationship preserves proportional comparison between harmonics.

**Dynamic Range:** The 0-100% scale accommodates the full theoretical range, but typical power systems exhibit much smaller harmonic percentages. A 10% THD might have individual harmonics of 5%, 3%, 2%, etc., resulting in relatively short bars.

**Minimum Visibility Threshold:** Very small harmonic values (below 0.5%) produce bars less than 1 pixel wide, which may not be visible. The implementation could add a minimum width threshold:

```cpp
// Ensure minimum visibility for non-zero harmonics
if (harmonicPercent > 0.0f && barWidth < 2) {
    barWidth = 2;  // Minimum visible width
}
```

---

## 16. Phasor Diagram Mathematics and Rendering

### 16.1 Phasor Diagram Theoretical Foundation

The Phase Angle screen renders a phasor diagram—a polar representation of voltage and current vectors that illustrates magnitude and phase relationships in the three-phase electrical system. This visualization is essential for understanding power factor, phase sequence, and load balance conditions.

The following diagram illustrates the phasor rendering pipeline from electrical measurements to screen display:

```
V/I Magnitudes & Angles → 6 Phasors → Normalize to 130px → Polar→Cartesian → Offset to (150,150) → Canvas (Lines + Arrowheads + Circles)
```

#### 16.1.1 Phasor Representation of Sinusoidal Quantities

In AC electrical systems, voltages and currents are sinusoidal quantities that can be mathematically represented as rotating vectors in the complex plane. A sinusoidal signal of the form:

$$v(t) = V_m \sin(\omega t + \phi)$$

Where:
- $V_m$ is the peak amplitude
- $\omega = 2\pi f$ is the angular frequency
- $\phi$ is the phase angle
- $t$ is time

Can be represented as a phasor (a complex number) in polar form:

$$\vec{V} = V_{rms} \angle \phi$$

Or equivalently in rectangular form:

$$\vec{V} = V_{rms}(\cos\phi + j\sin\phi)$$

The phasor magnitude equals the RMS value: $V_{rms} = V_m / \sqrt{2}$

#### 16.1.2 Three-Phase System Phasor Relationships

In a balanced three-phase system, the three voltage phasors are separated by 120 degrees:

$$\vec{V_R} = V \angle 0°$$
$$\vec{V_Y} = V \angle -120°$$
$$\vec{V_B} = V \angle -240° = V \angle +120°$$

The phase sequence (R-Y-B, also called positive sequence) means voltages reach their positive peaks in the order R, then Y, then B. The 120° separation ensures that the instantaneous sum of all three voltages is always zero:

$$\vec{V_R} + \vec{V_Y} + \vec{V_B} = 0$$

This property, known as the balanced system constraint, means the three phasors form a closed triangle when drawn tip-to-tail.

#### 16.1.3 Power Factor and Phase Angle Relationship

The power factor is directly related to the phase angle between voltage and current phasors for each phase:

$$PF = \cos(\theta_{V-I})$$

Where $\theta_{V-I}$ is the angle by which current leads or lags voltage. The phasor diagram visually represents this relationship—the angular separation between a voltage vector and its corresponding current vector directly indicates the power factor:

| Phase Angle | Power Factor | Load Type |
|-------------|--------------|-----------|
| 0° | 1.0 (Unity) | Purely resistive |
| +30° (I leads V) | 0.866 leading | Capacitive load |
| -30° (I lags V) | 0.866 lagging | Inductive load |
| +90° (I leads V by 90°) | 0.0 leading | Pure capacitor |
| -90° (I lags V by 90°) | 0.0 lagging | Pure inductor |

In a balanced three-phase system, voltage phasors are separated by 120 degrees, forming a symmetrical pattern. Current phasors may lead or lag their respective voltage phasors depending on the load's reactive characteristics. The phasor diagram displays all six quantities (three voltages and three currents) simultaneously, enabling visual assessment of system conditions.

### 16.2 Data Preparation and Phase Reference

The phasor data originates from the energy meter firmware, which calculates RMS magnitudes and phase angles from the sampled waveforms. The R-phase voltage (VR) serves as the reference phasor at 0 degrees, with all other angles measured relative to this reference.

The phase data preparation occurs in the firmware layer (`Core/Src/config_ui.c`):

```cpp
void get_phase_data(PhaseData* out) {
    // R-phase voltage is the reference phasor at 0 degrees
    out->rv.angle_deg = 0.0f;
    out->rv.magnitude = EM_PARAMS.VR;
    
    // Y-phase voltage referenced to VR
    out->yv.angle_deg = normalize_angle(EM_PARAMS.PhaseAngle_VR_VY);
    out->yv.magnitude = EM_PARAMS.VY;
    
    // B-phase voltage referenced to VR
    out->bv.angle_deg = normalize_angle(EM_PARAMS.PhaseAngle_VR_VB);
    out->bv.magnitude = EM_PARAMS.VB;
    
    // Current phasors referenced to VR
    out->ri.angle_deg = normalize_angle(EM_PARAMS.PhaseAngle_VR_IR);
    out->ri.magnitude = EM_PARAMS.IR;
    
    out->yi.angle_deg = normalize_angle(EM_PARAMS.PhaseAngle_VR_IY);
    out->yi.magnitude = EM_PARAMS.IY;
    
    out->bi.angle_deg = normalize_angle(EM_PARAMS.PhaseAngle_VR_IB);
    out->bi.magnitude = EM_PARAMS.IB;
}

// Normalize angle to 0-360 degree range
float normalize_angle(float angle) {
    while (angle > 360.0f) angle -= 360.0f;
    while (angle < 0.0f) angle += 360.0f;
    return angle;
}
```

### 16.3 Coordinate System Transformation Mathematics

Converting phasor data to screen coordinates requires a rigorous mathematical transformation from polar coordinates (magnitude and angle) to Cartesian pixel coordinates. This transformation involves several mathematical concepts that must be carefully implemented to produce an accurate visual representation.

#### 16.3.1 Polar to Cartesian Coordinate Conversion

In the mathematical polar coordinate system, a point is defined by a radius $r$ (distance from origin) and angle $\theta$ (measured counterclockwise from the positive x-axis). The equivalent Cartesian coordinates are:

$$x = r \cos(\theta)$$
$$y = r \sin(\theta)$$

These fundamental trigonometric relationships form the basis of the phasor rendering algorithm. For a phasor with magnitude $M$ and phase angle $\phi$ degrees:

$$x_{endpoint} = M \cos(\phi \times \frac{\pi}{180})$$
$$y_{endpoint} = M \sin(\phi \times \frac{\pi}{180})$$

The conversion factor $\frac{\pi}{180}$ transforms degrees to radians, as required by the C math library functions.

#### 16.3.2 Screen Coordinate System Differences

Computer graphics coordinate systems differ from mathematical conventions in a critical way: the Y-axis is inverted. In screen coordinates:
- **Origin (0,0)** is at the top-left corner of the display
- **Positive X** increases rightward (same as mathematical convention)
- **Positive Y** increases downward (opposite to mathematical convention)

This inversion requires modifying the y-coordinate calculation:

$$y_{screen} = y_{center} - r \sin(\theta)$$

The subtraction (rather than addition) compensates for the inverted Y-axis, ensuring that positive angles appear in the upper half of the diagram as expected from mathematical convention.

#### 16.3.3 Translation to Display Center

The phasor diagram is centered at pixel coordinates (150, 150) within a 300×300 pixel canvas area. This center point serves as the origin of the phasor coordinate system. The complete transformation from polar phasor coordinates to screen pixel coordinates is:

$$x_{pixel} = 150 + r_{scaled} \cos(\theta)$$
$$y_{pixel} = 150 - r_{scaled} \sin(\theta)$$

Where $r_{scaled}$ is the normalized phasor length in pixels (see magnitude normalization below).

#### 16.3.4 Magnitude Normalization Algorithm

Phasor magnitudes from the firmware represent actual electrical quantities (volts or amperes) that must be scaled to fit the display area. The normalization algorithm ensures the largest phasor fully utilizes the available display space while maintaining proportional relationships:

**Step 1:** Find the maximum magnitude across all voltage phasors and all current phasors separately:

$$M_{max,V} = \max(|V_R|, |V_Y|, |V_B|)$$
$$M_{max,I} = \max(|I_R|, |I_Y|, |I_B|)$$

**Step 2:** Calculate normalized lengths. The maximum vector length is 130 pixels (leaving margin within the 150-pixel radius display area):

$$L_V = \frac{M_V}{M_{max,V}} \times 130$$
$$L_I = \frac{M_I}{M_{max,I}} \times 130$$

**Mathematical Justification:** Normalizing voltage and current phasors independently ensures that both sets are visible even when their magnitudes differ by orders of magnitude (e.g., 230V and 5A). If normalized together, small current values would be invisible on a voltage-scaled display.

```cpp
// Implementation in PhaseAngleView::set_phase_data()
float max_v_mag = fmax(rv_mag, fmax(yv_mag, bv_mag));
float max_i_mag = fmax(ri_mag, fmax(yi_mag, bi_mag));

// Prevent division by zero for zero-magnitude cases
if (max_v_mag == 0) max_v_mag = 1.0f;
if (max_i_mag == 0) max_i_mag = 1.0f;

// Normalized lengths in pixels
float rv_len = (rv_mag / max_v_mag) * 130.0f;
float yv_len = (yv_mag / max_v_mag) * 130.0f;
float bv_len = (bv_mag / max_v_mag) * 130.0f;

float ri_len = (ri_mag / max_i_mag) * 130.0f;
float yi_len = (yi_mag / max_i_mag) * 130.0f;
float bi_len = (bi_mag / max_i_mag) * 130.0f;
```

#### 16.3.5 Complete Endpoint Calculation

Combining all transformations, the complete phasor endpoint calculation for a phasor with angle $\phi$ degrees and normalized length $L$ pixels:

```cpp
// Degree to radian conversion
const float DEG2RAD = M_PI / 180.0f;
float theta = angle_deg * DEG2RAD;

// Final screen coordinates
float end_x = 150.0f + length * cosf(theta);
float end_y = 150.0f - length * sinf(theta);  // Note the subtraction for Y-axis inversion
```

**Worked Example:**
- Y-phase voltage: magnitude = 230V (normalized to 130 px), angle = -120°
- $\theta = -120° \times \frac{\pi}{180} = -\frac{2\pi}{3}$ radians = -2.094 radians
- $\cos(-120°) = -0.5$, $\sin(-120°) = -0.866$
- $x_{pixel} = 150 + 130 \times (-0.5) = 150 - 65 = 85$
- $y_{pixel} = 150 - 130 \times (-0.866) = 150 + 112.6 = 262.6 \approx 263$
- The Y-phase voltage vector endpoint is at pixel (85, 263)

### 16.4 Arrowhead Rendering Mathematics

Each voltage phasor vector terminates with an arrowhead that indicates the direction of the phasor. The arrowhead is rendered as a filled triangle using TouchGFX Shape widgets. The mathematical derivation of the triangle vertices involves 2D rotation transformations.

#### 16.4.1 Arrowhead Geometry Definition

The arrowhead is an isoceles triangle with:
- **Tip point (P):** Located at the phasor endpoint
- **Base points (A and B):** Located behind the tip, perpendicular to the phasor direction
- **Size parameter (a):** Controls the arrowhead dimensions

In the TorTitan implementation, $a = 2.5 \times \sqrt{3} \approx 4.33$ pixels, creating an equilateral-proportioned arrowhead.

#### 16.4.2 Rotation Matrix Derivation

To position the arrowhead correctly for any phasor angle, we use the 2D rotation transformation. For a point $(x, y)$ rotated by angle $\theta$ about the origin:

$$\begin{bmatrix} x' \\ y' \end{bmatrix} = \begin{bmatrix} \cos\theta & -\sin\theta \\ \sin\theta & \cos\theta \end{bmatrix} \begin{bmatrix} x \\ y \end{bmatrix}$$

Expanding:
$$x' = x\cos\theta - y\sin\theta$$
$$y' = x\sin\theta + y\cos\theta$$

#### 16.4.3 Arrowhead Vertex Calculation

Consider an arrowhead aligned with the positive x-axis (θ = 0). The template vertices relative to the tip point are:
- **Tip:** $(0, 0)$
- **Base vertex A:** $(-a, -a)$ (behind and below)
- **Base vertex B:** $(-a, +a)$ (behind and above)

To rotate this template to match a phasor at angle $\theta$, we apply the rotation matrix. However, because screen coordinates have an inverted Y-axis, the rotation formulas are adjusted:

For base vertex A (template position $(-a, -a)$):
$$A_x = x_{tip} + [(-a)\cos\theta - (-a)\sin\theta] = x_{tip} - a\cos\theta - a\sin\theta$$
$$A_y = y_{tip} - [(-a)\sin\theta + (-a)\cos\theta] = y_{tip} + a\sin\theta - a\cos\theta$$

For base vertex B (template position $(-a, +a)$):
$$B_x = x_{tip} + [(-a)\cos\theta - (+a)\sin\theta] = x_{tip} - a\cos\theta + a\sin\theta$$
$$B_y = y_{tip} - [(-a)\sin\theta + (+a)\cos\theta] = y_{tip} + a\sin\theta + a\cos\theta$$

```cpp
// Arrowhead calculation from PhaseAngleView.cpp
float a = 2.5f * sqrtf(3.0f);  // ≈ 4.33 pixels

// Phasor tip coordinates (endpoint of vector)
float x = 150 + length * cos(theta);  // x_tip
float y = 150 - length * sin(theta);  // y_tip

// Arrowhead vertex A (one base corner)
float A_x = x - a * cos(theta) - a * sin(theta);
float A_y = y + a * sin(theta) - a * cos(theta);

// Arrowhead vertex B (other base corner)
float B_x = x - a * cos(theta) + a * sin(theta);
float B_y = y + a * sin(theta) + a * cos(theta);

// Create triangle shape with three vertices
touchgfx::AbstractShape::ShapePoint<float> shapePoints[3] = {
    {x, y},       // Tip (vertex 0)
    {A_x, A_y},   // Base vertex A (vertex 1)
    {B_x, B_y}    // Base vertex B (vertex 2)
};
arrowhead.setShape(shapePoints);
```

#### 16.4.4 Geometric Visualization

For a phasor pointing at angle θ:

```
                    • (x, y) = Tip
                   /\
                  /  \
                 /    \
                /      \
               •--------•
             A            B
```

The arrowhead rotates with the phasor direction, always pointing outward from the origin. The vertices A and B are positioned at distance $a\sqrt{2}$ from the tip, at angles $\theta + 135°$ and $\theta - 135°$ respectively.

### 16.5 Current Phasor Hollow Circle Mathematics

To visually distinguish current phasors from voltage phasors, current vectors display a small hollow circle at their tips instead of solid arrowheads. The circle positioning requires precise calculation to appear centered at the vector endpoint.

#### 16.5.1 Circle Positioning Strategy

The hollow circle is implemented using a TouchGFX Circle widget. However, circles are defined by their center point and radius. To position the circle so that the phasor tip touches the inner edge of the circle (rather than the center), the circle center must be offset along the phasor direction:

$$x_{circle} = x_{tip} + R \cos\theta$$
$$y_{circle} = y_{tip} - R \sin\theta$$

Where $R$ is the circle radius (4 pixels in the implementation).

#### 16.5.2 Geometric Interpretation

```
                    Phasor direction →
        
    Origin ─────────────────•──────○
                            │      │
                        Tip │      │ Circle center
                        (x_tip, y_tip)  (x_circle, y_circle)
                            
                            ←── R ──→
```

The circle center is displaced by one radius ($R = 4$ pixels) in the direction of the phasor. This positions the circle so that the phasor line terminates exactly at the inner circumference of the circle.

```cpp
// Current phasor circle positioning
const float R = 4.0f;  // Circle radius in pixels

// Calculate circle center offset from phasor tip
float cx = x + R * cos(theta);  // x = tip x-coordinate
float cy = y - R * sin(theta);  // y = tip y-coordinate (note Y-axis inversion)

// Configure the Circle widget
ri_tipCircle.setCenter(cx, cy);
ri_tipCircle.setRadius(R);
ri_tipCircle.setLineWidth(1.0f);  // Hollow circle (outline only)
ri_tipCircle.setArc(0, 360);      // Full circle
```

#### 16.5.3 Circle Widget Configuration

The hollow appearance is achieved by setting only a line width (outline) without a fill painter:

```cpp
// Setup in PhaseAngleView::setupScreen()
ri_tipCircle.setPosition(10, 137, 300, 300);  // Bounding box
ri_tipCircle.setCenter(150, 150);             // Initial center (updated dynamically)
ri_tipCircle.setRadius(4);
ri_tipCircle.setLineWidth(1.0f);              // 1-pixel outline
ri_tipCircle.setArc(0, 360);                  // Full 360° arc = complete circle

// Set outline color (red for R-phase current)
ri_tipCirclePainter.setColor(touchgfx::Color::getColorFromRGB(248, 40, 90));
ri_tipCircle.setPainter(ri_tipCirclePainter);

add(ri_tipCircle);  // Add to widget hierarchy
```

### 16.6 Color Coding and Visual Design

Consistent color coding enables quick identification of phases across the diagram:

| Phase | Element | Color | RGB Values |
|-------|---------|-------|------------|
| R | Voltage Vector | Red | (248, 40, 90) |
| R | Current Vector | Red | (248, 40, 90) |
| Y | Voltage Vector | Yellow | (246, 192, 0) |
| Y | Current Vector | Yellow | (246, 192, 0) |
| B | Voltage Vector | Cyan | (14, 186, 246) |
| B | Current Vector | Cyan | (14, 186, 246) |

The reference circle (centered at the origin with radius matching the longest phasor) provides a visual scale reference and is rendered in a neutral gray color.

### 16.7 Canvas Widget Usage

The phasor diagram extensively uses TouchGFX Canvas widgets for vector rendering:

**Line Widgets:** Render the X and Y axes and the phasor vectors themselves. Lines are configured with start and end coordinates, line width, and color.

**Circle Widgets:** Render the reference circle at the diagram center and the hollow circles at current phasor tips.

**Shape Widgets:** Render arrowhead triangles using three-vertex polygon definitions.

Canvas widgets are CPU-intensive because they require software rendering without DMA2D hardware acceleration. This makes the phasor diagram one of the most computationally demanding screens in the application. The design limits canvas widget usage to this single screen to maintain overall system performance.

---

## 17. Canvas Widget and Vector Drawing

### 17.1 Canvas Widget Architecture

Canvas widgets provide TouchGFX applications with the capability to render vector graphics—geometric shapes defined by mathematical descriptions rather than pixel bitmaps. This approach offers several advantages: shapes can be scaled without quality loss, complex curves can be represented with minimal memory, and dynamic graphics can be computed at runtime based on application data.

The TouchGFX framework provides three primary canvas widget types, each optimized for specific geometric primitives:

**Line Widget (`touchgfx::Line`):** Renders straight line segments between two points. Supports configurable line width, color, and optional anti-aliasing. Lines are defined by start and end coordinates within the widget's bounding rectangle.

**Circle Widget (`touchgfx::Circle`):** Renders circular arcs or complete circles. Supports configurable radius, line width, fill options, and arc angles for partial circles. Circles can be rendered as outlines (hollow) or filled shapes.

**Shape Widget (`touchgfx::Shape`):** Renders arbitrary polygons defined by vertex coordinates. Supports any number of vertices (minimum three for a triangle), enabling complex shapes such as arrowheads, custom indicators, or irregular regions.

### 17.2 Canvas Buffer Requirements

Canvas widgets require a dedicated rendering buffer for anti-aliased edge computation. This buffer stores intermediate rendering data during the rasterization process and must be sized appropriately for the most complex canvas widget visible at any time.

The canvas buffer is typically configured during HAL initialization:

```cpp
// Canvas buffer allocation in TouchGFXHAL.cpp or main initialization
static uint8_t canvasBuffer[3600];  // Size in bytes

// Configure the canvas renderer with the allocated buffer
CanvasWidgetRenderer::setupBuffer(canvasBuffer, sizeof(canvasBuffer));
```

Buffer size requirements depend on the complexity of rendered shapes. For the TorTitan phasor diagram (the most canvas-intensive screen), a 3600-byte buffer provides sufficient capacity. Undersized buffers result in rendering artifacts where complex portions of shapes are incompletely drawn.

### 17.3 Canvas Widget Configuration Pattern

Canvas widgets are configured programmatically, typically in the View or Container setup methods. The following pattern demonstrates Line widget configuration:

```cpp
void PhaseAngleView::setupCanvasWidgets() {
    // Configure X-axis line
    xAxisLine.setPosition(0, 150, 300, 1);  // x, y, width, height of bounding box
    xAxisLine.setStart(0, 0);                // Start point relative to bounding box
    xAxisLine.setEnd(300, 0);                // End point relative to bounding box
    xAxisLine.setLineWidth(1);               // Line thickness in pixels
    xAxisLine.setColor(Color::getColorFromRGB(128, 128, 128));  // Gray color
    xAxisLine.setVisible(true);
    add(xAxisLine);  // Add to widget hierarchy
    
    // Configure phasor vector line (dynamic positioning)
    phasorLine_R.setLineWidth(2);
    phasorLine_R.setColor(Color::getColorFromRGB(248, 40, 90));  // Red phase
    add(phasorLine_R);
}
```

### 17.4 Performance Considerations

Canvas widget rendering is computationally intensive because it relies entirely on software rasterization. Unlike bitmap widgets that can leverage DMA2D hardware acceleration for pixel copying, canvas widgets must compute each pixel through mathematical calculations. This performance characteristic has important implications for application design:

**CPU Load:** Each canvas widget visible on screen contributes to per-frame CPU load during rendering. Complex shapes with many vertices or wide anti-aliased lines require more computation.

**Rendering Time:** Canvas-heavy screens may exhibit longer frame rendering times, potentially affecting the achievable frame rate or leaving less CPU time for other application tasks.

**Strategic Placement:** The TorTitan design restricts canvas widget usage to the Phase Angle screen, which displays relatively static phasor diagrams. Frequently-updating screens (such as measurement displays) use simpler widget types that benefit from hardware acceleration.

### 17.5 Canvas Widgets in the TorTitan Project

The TorTitan project uses canvas widgets exclusively on the Phase Angle screen for phasor diagram rendering. The implementation includes:

| Widget Type | Usage | Count |
|-------------|-------|-------|
| Line | X and Y axes | 2 |
| Line | Phasor vectors (VR, VY, VB, IR, IY, IB) | 6 |
| Circle | Reference circle (diagram boundary) | 1 |
| Circle | Current phasor tip indicators | 3 |
| Shape | Voltage phasor arrowheads (triangles) | 3 |

This limited usage reflects the performance trade-offs inherent in canvas rendering. For simple geometric elements on other screens (rectangles, filled regions), the project uses Box widgets which render efficiently through DMA2D acceleration.

---

## 18. Memory Optimization and Performance Tuning

### 18.1 Framebuffer Strategy

The framebuffer configuration represents one of the most significant memory allocation decisions in any TouchGFX application. The TorTitan project employs single buffering with TFT controller optimization, a strategy that balances memory efficiency against rendering smoothness.

The configured strategy is `REFRESH_STRATEGY_OPTIM_SINGLE_BUFFER_TFT_CTRL`, which utilizes a single framebuffer and coordinates rendering with the display controller's refresh cycle. This approach requires approximately 300 KB for the framebuffer (320 × 480 × 2 bytes), compared to 600 KB that double buffering would require.

Double buffering, while providing tear-free rendering, was deemed infeasible for this project due to the STM32H750VB's limited internal RAM. The single-buffer approach occasionally produces minor visual artifacts during screen transitions, but these are acceptable given the memory constraints.

### 18.2 Memory Region Allocation

The STM32H750VB provides multiple RAM regions with different characteristics. The TorTitan project allocates these regions strategically based on access patterns and performance requirements:

| Memory Region | Base Address | Size | Usage in TorTitan |
|---------------|--------------|------|-------------------|
| DTCMRAM | 0x20000000 | 128 KB | Stack, frequently-accessed variables |
| RAM_D1 | 0x24000000 | 512 KB | Application data, FreeRTOS heap |
| RAM_D2 | 0x30000000 | 288 KB | **Framebuffer** (accessible by DMA2D) |
| RAM_D3 | 0x38000000 | 64 KB | Standby-retained data |
| ITCMRAM | 0x00000000 | 64 KB | Performance-critical code |
| External Flash | 0x90000000 | 8 MB | Code, fonts, images |

The framebuffer is placed in RAM_D2 specifically because this region is accessible by the DMA2D peripheral. Placing the framebuffer elsewhere would prevent hardware-accelerated rendering operations.

### 18.3 Performance Optimization Techniques

The TorTitan implementation incorporates several performance optimization techniques that collectively enable smooth 60 fps operation:

**Targeted Widget Invalidation:** The most impactful optimization involves invalidating only changed widgets rather than entire screens. A call to `widget.invalidate()` marks only that widget's region for redrawing, while `invalidate()` at the screen level marks the entire display. The rendering engine only processes invalidated regions, dramatically reducing per-frame workload.

```cpp
// Efficient: Only redraws the voltage text widget
vr_text.invalidate();

// Inefficient: Redraws the entire screen
invalidate();
```

**Data Update Throttling:** The Model layer throttles data broadcasts to approximately 2 Hz (every 30 frames at 60 fps). This rate is sufficient for human perception while reducing the number of widget updates and invalidations per second.

**Transition-Free Navigation:** Screen transitions (slide, fade, zoom effects) require temporary double buffering and significantly increase CPU load. The TorTitan project uses `NoTransition` navigation exclusively, eliminating this overhead.

**Minimal Canvas Usage:** Canvas widgets are restricted to the Phase Angle screen, preventing software rendering overhead from affecting frequently-updated measurement screens.

**Pre-allocated Buffers:** All wildcard text buffers and data structures are statically allocated, avoiding dynamic memory allocation overhead and fragmentation risks during operation.

**DMA2D Hardware Acceleration:** Widget rendering leverages the ChromART (DMA2D) hardware accelerator for block copy and fill operations. This offloads pixel manipulation from the CPU to dedicated hardware.

**Cache Management:** Proper cache coherency is maintained through strategic use of `SCB_CleanInvalidateDCache()` before DMA transfers, ensuring data consistency between CPU cache and DMA-accessible memory.

### 18.4 Practices to Avoid

Certain implementation patterns can severely degrade performance and should be avoided:

**Screen Transitions:** Never use animated screen transitions (slide, fade, etc.) as they require double buffering and substantial CPU resources.

**Full-Screen Invalidation:** Avoid invalidating entire screens every frame. Even on screens with many updating widgets, individual widget invalidation is more efficient.

**Large Transparent Images:** Images with alpha transparency require software blending, which is significantly slower than opaque image rendering. Minimize use of transparency in image assets.

**Canvas on Updating Screens:** Do not place canvas widgets on screens that update frequently. The combination of software rendering and frequent invalidation creates unsustainable CPU load.

**Excessive Graph Elements:** Each Graph widget with data points consumes memory and rendering time. Limit the number of simultaneous graphs and consider reducing point counts if performance issues arise.

---

## 19. Common Issues and Debugging Procedures

### 19.1 HardFault During Screen Transitions

**Symptom Description:** The device resets, freezes, or enters an infinite loop when navigating between screens. The debugger may show execution stopped in the HardFault handler.

**Root Cause Analysis:** HardFault during navigation typically indicates memory corruption, stack overflow, or null pointer dereference. In the TouchGFX context, these issues commonly arise from:

*Stack Overflow:* The TouchGFX task stack (currently 2176 bytes) may be insufficient for screens with complex widget hierarchies or deep function call chains. Stack overflow corrupts adjacent memory, often causing HardFault on the next function call or return.

*Presenter Data Race:* When the Model broadcasts data during a screen transition, the deactivating Presenter may attempt to forward data to a View that is being destroyed. This can result in null pointer dereference.

*Widget Lifetime Issues:* Storing pointers to widgets that are destroyed during screen transitions and subsequently accessing those pointers causes undefined behavior.

*Buffer Overflow:* Wildcard text buffers with insufficient size can overwrite adjacent memory, corrupting stack frames or other data structures.

**Diagnostic Procedure:**
1. Enable FreeRTOS stack overflow checking by setting `configCHECK_FOR_STACK_OVERFLOW` to 2 in FreeRTOSConfig.h
2. Implement and register the stack overflow hook to identify the offending task
3. If stack overflow is confirmed, increase the TouchGFX task stack size
4. If not stack-related, add null pointer checks in Presenter data forwarding methods
5. Review wildcard buffer sizes against maximum possible string lengths

**Resolution:** Increase stack allocation if overflow is confirmed. Add defensive null checks in data forwarding paths. Verify all buffer sizes are adequate.

### 19.2 Font and Character Display Issues

**Symptom Description:** Text appears blank, displays as boxes or rectangles, or shows garbled characters.

**Root Cause Analysis:** Font rendering issues indicate mismatches between expected characters and compiled typography data. Specific causes include:

*Missing Characters:* The Typography definition in TouchGFX Designer specifies which characters are compiled. Characters outside this range render as blank.

*Wrong Typography Assignment:* TextArea widgets may reference incorrect Typography IDs, causing the renderer to use font data that doesn't match the displayed text.

*Regeneration Required:* Adding characters to a Typography definition requires code regeneration. Until regeneration, the font data doesn't include new characters.

**Diagnostic Procedure:**
1. Open TouchGFX Designer and navigate to Texts → Typographies
2. Verify the Typography used by the problematic TextArea includes all required characters
3. Check for special characters (°, %, ±, units symbols) in dynamic text that may not be included
4. Regenerate code (F4) after modifying Typography definitions
5. Verify the correct Typography is assigned to the TextArea widget

**Resolution:** Add missing characters to the Typography character range. Regenerate code. Verify Typography assignments in the Designer.

### 19.3 Display Lag and Flickering

**Symptom Description:** The display visibly updates in stages, text flickers during updates, or the interface feels sluggish.

**Root Cause Analysis:** Visual lag and flickering indicate rendering inefficiency where too much screen content is being redrawn each frame. Causes include:

*Excessive Invalidation:* Invalidating entire screens or large widget regions every frame forces complete redraws.

*Update Rate Mismatch:* Updating widget content every frame (60 Hz) when data only changes at 2 Hz creates unnecessary rendering work.

*Canvas Widget Overhead:* Canvas widgets on frequently-updating screens consume excessive CPU during rendering.

*Graph Performance:* Large numbers of graph data points or multiple simultaneous graphs increase rendering time.

**Diagnostic Procedure:**
1. Review invalidation patterns in View update methods—ensure only changed widgets are invalidated
2. Verify data throttling is implemented in the Model layer
3. Identify whether lag correlates with specific screens (indicating screen-specific issues)
4. Check if canvas widgets are present on affected screens
5. Review graph configurations for excessive point counts

**Resolution:** Implement targeted invalidation. Add or verify data throttling. Move canvas widgets to less frequently-updated screens if possible. Reduce graph point counts.

### 19.4 Touch Input Issues

**Symptom Description:** Touch input is unresponsive or coordinates are incorrect.

**Root Cause Analysis:** In the TorTitan project, touch functionality is intentionally disabled. The `STM32TouchController::sampleTouch()` method returns `false`, indicating no touch input. This is by design, as the product uses joystick navigation.

If touch capability is required in future variants, implementation would involve:

*Touch Controller Implementation:* The sampleTouch method must communicate with the touch IC via I2C or SPI and return valid coordinates.

*Coordinate Mapping:* Touch IC coordinates may not match display coordinates. Transformation matrices handle rotation, mirroring, and scaling.

*Calibration:* Physical variations between touch panels and displays may require per-unit calibration.

### 19.5 Graph Performance Degradation

**Symptom Description:** Screens containing graphs exhibit poor frame rate or visible rendering delays.

**Root Cause Analysis:** Graph widgets involve significant rendering work, particularly when many data points are present or multiple graphs update simultaneously.

*Point Count:* Each data point requires processing during graph rendering. The 64-point configuration represents a balance between visual fidelity and performance.

*Simultaneous Updates:* Updating all three phase graphs simultaneously triples the rendering workload.

*Frequent Clearing:* Clearing and repopulating graph data every frame is inefficient.

**Diagnostic Procedure:**
1. Verify graph updates are throttled to the data update rate (not every frame)
2. Check if graph data is cleared unnecessarily on each update
3. Consider reducing point count from 64 to 32 if performance is critical
4. Evaluate whether all visible graphs need simultaneous updates

**Resolution:** Implement update throttling. Avoid unnecessary graph clearing. Consider point count reduction for performance-critical deployments.

### 19.6 Additional Documented Issues

| Issue | Typical Cause | Resolution |
|-------|---------------|------------|
| Display blank after boot | VSYNC task not started | Verify task creation in freertos.c |
| Data shows "0" constantly | Model::tick() not receiving firmware data | Check xTaskNotifyWait and sender task |
| Navigation becomes stuck | Joystick event queue overflow or misconfiguration | Verify queue creation and ISR event posting |
| Auto-scroll not functioning | Tick counter logic error | Review counter increment and threshold comparison |
| Values display "--" unexpectedly | NaN or infinity from firmware calculations | Debug EM_PARAMS data source |
| DMA2D visual artifacts | CPU cache not synchronized with DMA | Add SCB_CleanInvalidateDCache() before transfers |
| Alarm screen appears erroneously | Alarm flag bouncing without edge detection | Implement previous/current state comparison |

### 19.7 Debugging Tools and Techniques

**STM32CubeIDE Debugger:** Set breakpoints in View and Presenter code to trace data flow and widget updates. The debugger provides full visibility into execution state.

**Live Expressions:** Configure watch expressions for EM_PARAMS structure members to monitor measurement data in real-time during debugging.

**FreeRTOS-Aware Debugging:** STM32CubeIDE's RTOS-aware features display task states, stack usage, and queue contents. Essential for diagnosing task-related issues.

**Serial Debug Output:** Add UART printf statements in Model::tick() to trace data flow when visual debugging is impractical.

**TouchGFX Simulator:** Test UI logic independently of hardware using the PC simulator. Note that backend integration is not available in simulation.

---

## 20. Touch Controller Integration

### 20.1 Current Implementation Status

The TorTitan product uses joystick-based navigation and does not implement touch screen functionality. The touch controller interface exists in the codebase but is disabled:

```cpp
// TouchGFX/target/STM32TouchController.cpp
bool STM32TouchController::sampleTouch(int32_t& x, int32_t& y) {
    // Touch functionality disabled - joystick-only device
    return false;
}
```

The return value of `false` indicates to the TouchGFX framework that no touch input is available, causing touch-related event processing to be skipped entirely.

### 20.2 Touch Implementation Requirements

If future product variants require touch functionality, the following implementation work would be necessary:

**Hardware Interface:** Implement communication with the touch controller IC (typically GT911, FT5336, or similar) via I2C or SPI. This includes initialization sequence, interrupt handling, and coordinate reading.

**Driver Integration:** Complete the STM32TouchController class implementation:

```cpp
void STM32TouchController::init() {
    // Initialize touch controller hardware interface
    // Configure I2C or SPI peripheral
    // Send initialization commands to touch IC
    // Configure touch IC for appropriate sampling rate
    // Set up interrupt pin if using interrupt-driven mode
}

bool STM32TouchController::sampleTouch(int32_t& x, int32_t& y) {
    // Check if touch is active
    if (!touch_ic_has_touch()) {
        return false;
    }
    
    // Read raw coordinates from touch IC
    uint16_t raw_x, raw_y;
    touch_ic_read_coordinates(&raw_x, &raw_y);
    
    // Transform coordinates to match display orientation
    // Apply calibration if required
    x = transform_x(raw_x);
    y = transform_y(raw_y);
    
    return true;
}
```

**Coordinate Transformation:** Touch panel coordinates often require transformation to match display coordinates. This may include rotation (if panel is mounted rotated), mirroring (if coordinates are inverted), and scaling (if touch resolution differs from display resolution).

**Calibration System:** Production variation between touch panels and displays may require per-device calibration. A calibration routine presents touch targets at known positions and computes transformation parameters from the resulting touch coordinates.

### 20.3 Joystick Navigation Architecture

In lieu of touch, the TorTitan implements comprehensive joystick navigation. The joystick hardware consists of a 5-way navigation switch connected to GPIO pins, with the following signal mapping:

| Direction | GPIO Pin | Active State |
|-----------|----------|--------------|
| UP | PE0 | Low (with pull-up) |
| DOWN | PE1 | Low (with pull-up) |
| LEFT | PE4 | Low (with pull-up) |
| RIGHT | PE3 | Low (with pull-up) |
| CENTER | PE5 | Low (with pull-up) |

The joystick driver (`Core/Src/Joystick.c`) implements:

**Debouncing:** 50ms debounce period to filter mechanical switch bounce

**Long Press Detection:** Separate events for short press (< 1000ms) and long press (≥ 1000ms)

**Event Queue:** FreeRTOS queue for thread-safe event delivery from GPIO ISR to TouchGFX task

**Auto-Sleep Coordination:** Joystick activity resets the display sleep timer

The joystick effectively substitutes for touch navigation, with directional presses handling screen navigation and center press handling selection/confirmation actions.

---

## 21. Backend Integration Architecture

### 21.1 Integration Overview

The TouchGFX graphics layer integrates with the energy meter firmware through a well-defined interface based on shared data structures and FreeRTOS synchronization primitives. This architecture enables the graphics subsystem to display real-time measurement data while maintaining clear separation between the measurement firmware and presentation logic.

The integration operates on a producer-consumer model where the energy measurement task produces data by updating shared structures, and the TouchGFX task consumes this data by reading the structures during each frame cycle. Synchronization ensures data consistency without blocking either task unnecessarily.

The following diagram illustrates the integration architecture:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Core Firmware (Energy Measurement)                                      │
│  ┌────────────┐   ┌────────────────┐   ┌─────────────────────────────┐  │
│  │ ADC/SPI    │──▶│  EM_PARAMS     │──▶│ xTaskNotify()               │  │
│  │ Sampling   │   │  Calculation   │   │ to TouchGFX task            │  │
│  └────────────┘   └────────────────┘   └─────────────┬───────────────┘  │
│                                                       │                  │
├───────────────────────────────────────────────────────┼──────────────────┤
│  TouchGFX Layer                                       ↓                  │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ Model::tick()                                                        ││
│  │   • xTaskNotifyWait() receives notification                         ││
│  │   • Reads EM_PARAMS structure (extern)                               ││
│  │   • Reads waveform arrays Vr[256], Vy[256], Vb[256] (extern)       ││
│  │   • Prepares screen-specific data structures                         ││
│  │   • Calls modelListener->set_xxx_data() methods                     ││
│  └─────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

### 21.2 Shared Data Structures

The firmware and graphics layers share several data structures and arrays through extern declarations in the Model implementation file. These shared elements represent the primary data interface between subsystems:

```cpp
// External declarations in Model.cpp

// Primary measurement parameters structure
// Contains all calculated electrical values
extern PARAMS_t EM_PARAMS;

// Voltage waveform samples (256 points per cycle)
extern float Vr[256], Vy[256], Vb[256];       // Line-to-Neutral
extern float Vry[256], Vyb[256], Vbr[256];    // Line-to-Line

// Current waveform samples
extern float Ir[256], Iy[256], Ib[256];

// Configuration structure
extern EM_CONFIG_t EM_CONFIG;

// Display power management
extern bool isDisplayOn;
extern void Sleep_Display();
extern void Wake_Display();

// Inter-task communication
extern QueueHandle_t joystick_queue;
extern uint8_t trigger_alarm;
```

The `PARAMS_t` structure contains the complete set of calculated electrical parameters including RMS voltages, currents, powers, energy accumulators, frequency, power factor, phase angles, and harmonic data. This structure is updated by the energy measurement task after each measurement cycle completes.

### 21.3 Synchronization Mechanism

The synchronization between the measurement task and TouchGFX task uses FreeRTOS task notifications, a lightweight mechanism that signals data availability without requiring semaphores or mutexes.

**Notification Flow:**
1. The energy measurement task completes a calculation cycle
2. Measurement task calls `xTaskNotify()` to signal the TouchGFX task
3. During `Model::tick()`, TouchGFX task calls `xTaskNotifyWait()`
4. If notification received, Model reads updated data and broadcasts to listeners

```cpp
// In Model::tick()
uint32_t notification_value;
if (xTaskNotifyWait(0, 0xFFFFFFFF, &notification_value, 0) == pdTRUE) {
    // New data available - process and broadcast
    if (data_throttle_counter >= 30) {
        broadcast_data_to_active_screen();
        data_throttle_counter = 0;
    }
}
data_throttle_counter++;
```

The non-blocking `xTaskNotifyWait()` call (timeout parameter 0) ensures the TouchGFX task never stalls waiting for data. If no notification is pending, the task continues with frame rendering using previously available data.

### 21.4 Data Flow Direction

**Downstream (Firmware → Display):** Measurement data flows from the EM_PARAMS structure through the Model layer to active Presenters and Views. This unidirectional flow ensures display content always reflects the current measurement state.

**Upstream (Display → Firmware):** Configuration changes initiated through the settings screens flow upstream. The Presenter writes new values directly to the EM_CONFIG structure and invokes the non-volatile storage save function. After saving, it triggers navigation via the View's custom action:

```cpp
// Settings configuration update example
void Settings_PT_RatioPresenter::save_configuration(uint32_t primary, uint32_t secondary) {
    // Update shared configuration structure
    EM_CONFIG.PT_Primary = primary;
    EM_CONFIG.PT_Secondary = secondary;
    
    // Persist to non-volatile storage
    nv_storage_save(&EM_CONFIG);
    
    // Confirm success to user via View's custom action
    getView().takeToSaveSuccess();  // Triggers "Change screen" interaction to success screen
}
```

### 21.5 Thread Safety Considerations

The integration design carefully manages thread safety through architectural patterns rather than explicit locking:

**Single-Writer Principle:** The EM_PARAMS structure is written exclusively by the energy measurement task. The TouchGFX task only reads from this structure, eliminating write-write conflicts.

**Atomic Data Consistency:** Individual measurements (single float values) are naturally atomic on the Cortex-M7 architecture. For complex data that spans multiple memory locations, the notification mechanism provides implicit synchronization—the TouchGFX task reads data only after the measurement task signals completion.

**Queue-Based Events:** Joystick events use a FreeRTOS queue, which provides built-in thread safety for event passing from ISR context to task context.

---

## 22. Project-Specific Logic and Reusable Components

### 22.1 Auto-Scaling Value Display

The TorTitan interface displays electrical measurements that can span many orders of magnitude. A current transformer with a 5000:5 ratio can produce readings from milliamps to megamps depending on system configuration. The auto-scaling display pattern automatically selects appropriate engineering unit prefixes:

```cpp
void formatWithAutoScale(float value, char* value_buffer, char* unit_buffer) {
    const char* prefix = "";
    const char* base_unit = "V";  // Or "A", "W", etc.
    
    if (fabs(value) >= 1e9) {
        value /= 1e9;
        prefix = "G";
    } else if (fabs(value) >= 1e6) {
        value /= 1e6;
        prefix = "M";
    } else if (fabs(value) >= 1e3) {
        value /= 1e3;
        prefix = "K";
    }
    
    snprintf(value_buffer, BUFFER_SIZE, "%.2f", value);
    snprintf(unit_buffer, UNIT_SIZE, "%s%s", prefix, base_unit);
}
```

This pattern is applied consistently across voltage, current, power, and energy displays, ensuring users see appropriately-scaled values regardless of transformer ratios.

### 22.2 Invalid Data Handling

Measurement calculations can produce invalid results (NaN or infinity) due to sensor errors, divide-by-zero conditions, or other exceptional circumstances. The interface must handle these gracefully:

```cpp
void displayMeasurement(float value, Unicode::UnicodeChar* buffer, int size) {
    if (isnan(value) || isinf(value)) {
        // Display placeholder for invalid data
        Unicode::fromUTF8((uint8_t*)"--", buffer, size);
    } else {
        // Normal formatting
        char temp[50];
        snprintf(temp, sizeof(temp), "%.2f", value);
        Unicode::fromUTF8((uint8_t*)temp, buffer, size);
    }
}
```

The "--" placeholder clearly indicates to users that data is unavailable, distinguishing from a legitimate zero reading.

### 22.3 Display Sleep/Wake Management

To conserve power and extend display backlight life, the interface implements automatic sleep after a configurable period of user inactivity:

```cpp
// In Model::tick()
void Model::tick() {
    static uint32_t idle_counter = 0;
    const uint32_t SLEEP_THRESHOLD = 1500;  // ~25 seconds at 60fps
    
    if (joystick_activity_detected) {
        idle_counter = 0;
        if (!isDisplayOn) {
            Wake_Display();
            isDisplayOn = true;
        }
    } else {
        idle_counter++;
        if (idle_counter >= SLEEP_THRESHOLD && isDisplayOn) {
            Sleep_Display();
            isDisplayOn = false;
            // Optionally navigate to Black_Screen
            modelListener->goto_black_screen();
        }
    }
}
```

The `Sleep_Display()` and `Wake_Display()` functions (implemented in firmware) control the LCD backlight PWM. The Black_Screen screen provides a minimal display state that still allows wake-on-input.

### 22.4 Screen State Tracking

The Model maintains awareness of the currently active screen to enable conditional data updates and screen-specific behavior:

```cpp
// Screen enumeration for state tracking
typedef enum {
    VAF_SCREEN,
    VOLTAGE_LN_SCREEN,
    VOLTAGE_LL_SCREEN,
    CURRENT_SCREEN,
    ACTIVE_POWER_SCREEN,
    REACTIVE_POWER_SCREEN,
    APPARENT_POWER_SCREEN,
    PHASE_ANGLE_SCREEN,
    // ... additional screens
    SETTINGS_MENU_SCREEN,
    BLACK_SCREEN
} eScreen;

// Model member variable
eScreen current_screen;
```

This state enables optimizations where only data relevant to the active screen is processed and broadcast. Screens in settings mode may skip measurement data processing entirely.

### 22.5 Reusable Container Summary

The following custom containers provide reusable visualization capabilities:

**Static_Graph:** Three-phase waveform display suitable for any voltage or current visualization screen. Accepts waveform sample arrays and renders synchronized phase displays with color-coded traces.

**Distortion_Graph:** Horizontal bar chart for harmonic spectrum visualization. Reusable for any per-harmonic data display, not limited to voltage or current harmonics.

**Phase_Angle_Screens:** Self-contained phasor diagram with configurable phase data input. Can be reused for any application requiring polar vector visualization.

**Distortion_Screen:** THD summary display with phase selection. Provides a template for any three-value comparison display with selection capability.

---

## 23. Build Generation and Code Regeneration

### 23.1 Code Generation Sources

The TorTitan project utilizes two code generation tools that produce complementary but distinct outputs:

**TouchGFX Designer:** Generates graphics framework code including screen base classes, widget hierarchies, font data, image assets, and text databases. Generation is triggered manually through the Designer interface (F4 key) or by opening the project file.

**STM32CubeMX:** Generates hardware abstraction layer code, peripheral initialization, FreeRTOS configuration, and clock setup. Generation is triggered through the CubeMX interface embedded in STM32CubeIDE.

These tools generate code into different directories and generally do not conflict, but awareness of their behavior is essential for maintaining code integrity.

| Tool | Output Directory | Trigger |
|------|------------------|---------|
| TouchGFX Designer | `TouchGFX/generated/` | F4 or Generate button |
| TouchGFX Designer | `TouchGFX/gui/` (stubs only, first-time) | New screen/container creation |
| STM32CubeMX | `Core/`, `Drivers/`, `Middlewares/` | "Generate Code" in .ioc |
| STM32CubeMX | `TouchGFX/target/generated/` | Peripheral changes |

### 23.2 Safe Regeneration Workflow

Code regeneration risks overwriting custom modifications if not performed carefully. The following workflow minimizes risk:

**Step 1: Commit Current State**
Before any regeneration operation, commit all current changes to version control:
```bash
git add -A
git commit -m "Pre-regeneration checkpoint"
```

**Step 2: Perform Generation**
Execute the code generation in the appropriate tool (TouchGFX Designer or CubeMX).

**Step 3: Review Changes**
Examine the generated changes using version control diff:
```bash
git diff
git status
```

**Step 4: Verify No User Code Loss**
Confirm that files in `gui/src/` and `gui/include/gui/` were not modified. These directories contain user code that should persist across regeneration.

**Step 5: Restore If Necessary**
If user code was unexpectedly modified, restore from version control:
```bash
git checkout -- gui/src/
git checkout -- gui/include/gui/
```

**Step 6: Build and Test**
Rebuild the project and verify functionality before committing regenerated code.

### 23.3 Overwrite Risk Analysis

Understanding which files are safe to regenerate versus which contain custom code is critical:

| File Path | Generator | Overwrite Safe? | Notes |
|-----------|-----------|-----------------|-------|
| `generated/gui_generated/**` | TouchGFX | ✅ Yes | Always regenerated |
| `generated/fonts/**` | TouchGFX | ✅ Yes | Font compilations |
| `generated/images/**` | TouchGFX | ✅ Yes | Image compilations |
| `generated/texts/**` | TouchGFX | ✅ Yes | Text database |
| `gui/src/**` | TouchGFX | ⚠️ Stubs only | New files created, existing preserved |
| `gui/include/gui/**` | TouchGFX | ⚠️ Stubs only | New files created, existing preserved |
| `target/generated/**` | CubeMX | ✅ Yes | HAL regeneration |
| `Core/Src/main.c` | CubeMX | ⚠️ Partial | Preserves USER CODE blocks |
| `Core/Src/freertos.c` | CubeMX | ⚠️ Partial | Preserves USER CODE blocks |

### 23.4 STM32CubeMX USER CODE Blocks

STM32CubeMX preserves code placed within designated USER CODE blocks during regeneration. Code outside these blocks is overwritten:

```c
/* USER CODE BEGIN Includes */
#include "my_custom_header.h"  // This is preserved
/* USER CODE END Includes */

// Code placed here WILL BE DELETED during regeneration

/* USER CODE BEGIN 0 */
// Custom code here is preserved
void my_custom_function(void) {
    // Implementation preserved
}
/* USER CODE END 0 */
```

**Critical Rule:** All custom modifications to CubeMX-generated files must reside within USER CODE blocks. Any code placed outside these markers will be lost during the next code generation cycle.

### 23.5 Build System Commands

The project builds through STM32CubeIDE using the integrated build system:

| Operation | Method | Notes |
|-----------|--------|-------|
| Full Build | Project → Build All (Ctrl+B) | Compiles all sources, links |
| Clean Build | Project → Clean, then Build | Removes all intermediates first |
| TouchGFX Regenerate | Open .touchgfx → F4 | Regenerates framework code only |
| CubeMX Regenerate | Open .ioc → Generate Code | Regenerates HAL code |

---

## 24. Critical Files and Overwrite Protection

### 24.1 User-Modified Files Inventory

The following files contain project-specific custom code that must be protected from accidental overwrite. These represent the core intellectual property of the TouchGFX implementation:

| File | Purpose | Criticality |
|------|---------|-------------|
| `gui/src/model/Model.cpp` | Backend integration, tick logic, joystick handling, data throttling | Critical |
| `gui/include/gui/model/Model.hpp` | Model state variables, screen enumeration | Critical |
| `gui/include/gui/model/ModelListener.hpp` | All listener interface methods | Critical |
| `gui/src/<screen>/View.cpp` | Screen display logic, widget updates | Critical (per screen) |
| `gui/src/<screen>/Presenter.cpp` | Navigation logic, data routing | Critical (per screen) |
| `gui/src/containers/*.cpp` | Graph, Phasor, Distortion implementations | Critical |
| `gui/include/gui/containers/*.hpp` | Container interfaces | Critical |
| `target/STM32TouchController.cpp` | Touch driver (disabled but customized) | Moderate |
| `target/TouchGFXHAL.cpp` | Display HAL, canvas buffer setup | Critical |
| `Core/Src/config_ui.c` | Phasor angle calculations | Critical |
| `Core/Inc/config_ui.h` | PhaseData structures | Critical |
| `Core/Src/Joystick.c` | Joystick driver, debouncing, queue | Critical |
| `Core/Inc/Joystick.h` | Joystick types and definitions | Critical |

### 24.2 Auto-Generated Files (Safe to Regenerate)

The following file patterns are auto-generated and will be regenerated during normal development workflow:

| Path Pattern | Generator | Regeneration Frequency |
|--------------|-----------|----------------------|
| `generated/gui_generated/**` | TouchGFX Designer | On any Designer change |
| `generated/fonts/**` | TouchGFX Designer | On typography change |
| `generated/images/**` | TouchGFX Designer | On image asset change |
| `generated/texts/**` | TouchGFX Designer | On text resource change |
| `target/generated/**` | STM32CubeMX | On peripheral change |
| `Drivers/**` | STM32CubeMX | On HAL update |
| `Middlewares/ST/**` | STM32CubeMX | On middleware update |

### 24.3 Fundamental Protection Rules

Two fundamental rules govern file modifications in this project:

**Rule 1: Never Modify Generated Files**
Files within the `generated/` directory hierarchy and `target/generated/` are completely regenerated during code generation. Any manual modifications will be lost without warning. If behavior changes are required for generated code, implement them through inheritance in the user code directories.

**Rule 2: Keep User Code in Designated Locations**
All custom application logic must reside in the `gui/src/`, `gui/include/gui/`, and `Core/` directories. Within `Core/`, custom code must be placed inside USER CODE blocks to survive CubeMX regeneration.

### 24.4 Version Control Strategy

Effective version control practices protect against accidental code loss:

**Pre-Regeneration Commits:**
```bash
git add -A && git commit -m "Checkpoint before regeneration"
```

**Post-Regeneration Review:**
```bash
git diff                    # Review all changes
git diff gui/src/           # Focus on user code directory
```

**Restoration Procedure:**
```bash
git checkout -- gui/src/    # Restore if user files modified
git checkout HEAD~1 -- <file>  # Restore specific file from previous commit
```

**Branch Strategy:** Consider using feature branches for significant TouchGFX Designer changes, merging only after verifying no user code disruption.

### 24.5 Key File Location Reference

This quick reference summarizes the location of major functional components:

| Functional Component | File Location |
|---------------------|---------------|
| Project Configuration | `TorTitan_withStm32CubeIde.touchgfx` |
| Central Data Model | `TouchGFX/gui/src/model/Model.cpp` |
| Event Interface | `TouchGFX/gui/include/gui/model/ModelListener.hpp` |
| Screen Views | `TouchGFX/gui/src/<screen_name>/` |
| Custom Containers | `TouchGFX/gui/src/containers/` |
| Phasor Calculations | `Core/Src/config_ui.c` |
| Joystick Driver | `Core/Src/Joystick.c` |
| Display HAL | `TouchGFX/target/TouchGFXHAL.cpp` |
| FreeRTOS Configuration | `Core/Inc/FreeRTOSConfig.h` |
| Memory Layout | `STM32H750VBTX_FLASH.ld` |
| Build Configuration | `TouchGFX/config/gcc/app.mk` |
| Application Entry | `TouchGFX/App/app_touchgfx.c` |

---

## Document Conclusion

This design document provides comprehensive technical documentation of the TouchGFX graphics framework implementation within the TorTitan Energy Meter project. The document covers architectural decisions, implementation patterns, and operational procedures necessary for ongoing development and maintenance of the graphical user interface.

Key areas addressed include the Model-View-Presenter communication architecture, dynamic text rendering with Unicode handling, specialized visualization components for waveforms and phasor diagrams, memory optimization strategies for resource-constrained embedded deployment, and code management practices that protect custom implementation while leveraging code generation tools.

Developers working on this project should use this document as a reference for understanding established patterns and as a guide for extending functionality while maintaining consistency with existing implementation approaches.

---

*End of TouchGFX Graphics Framework Design Document*
