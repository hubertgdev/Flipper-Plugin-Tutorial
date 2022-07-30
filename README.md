# Flipper-Plugin
Tutorial on how to build a basic "Hello world" plugin for Flipper Zero.

This tutorial includes: 

- Sourcecode template for a custom flipper plugin!
- A step by step story on how to create the base of a custom flipper plugin. (There are many other ways of creating apps for flipper, this example is based on the existing snake app)

The tutorial was written during development of [flappybird for flipper](https://github.com/DroomOne/flipperzero-firmware/tree/dev/applications/flappy_bird). 

![Screenshot](/img/qFlipper_zp57pxf0Dv.gif)

# Hello World - The story
__This is the step-by-step story version of the tutorial. You can skip this and directly continue to the sourcecode if you know what your doing. Make sure you don't forget to add the application to the makefile, and register its functions in `applications.c` (see chapter: Building the firmware + plugin)__



In this tutorial a simple hello world plugin is added to flipper. The goal is to render something in the screen, and make the buttons move that object. In this case it will be the classic "Hello World" text. 

## Downloading the firmware
1. Clone or download [flipperzero-firmware](https://github.com/flipperdevices/flipperzero-firmware). 
```sh 
git clone https://github.com/flipperdevices/flipperzero-firmware
```
2. Create a folder for the custom plugin in `flipperzero-firmware/applications/`. For the hello-world app, this will be: `hello_world`. 
```sh
mkdir flipperzero-firmware/applications/hello_world
```
3. Create a new source file in the newly created folder. The file name has to match the name of the folder. 

```sh
touch flipperzero-firmware/applications/hello_world/hello_world.c
```


## Includes
First, add the following includes needed for this plugin:

```c
#include <furi.h>
#include <gui/gui.h>
#include <input/input.h>
#include <stdlib.h>
```

## Plugin Main 
For flipper to activate the plugin, a main function for the plugin has to be added. Following the naming convention of existing flipper plugins, this needs to be: `hello_world_app`. 

- Create an `int32_t hello_world_app(void* p)` function that will function as the entry of the plguin. 

For the plugin to keep track of what actions have been executed, we create a messagequeue. 
- A by calling `osMessageQueueNew` we create a new `osMessageQueueId_t` that keeps track of events. 

The view_port is used to control the canvas (display) and userinput from the hardware. In order to use this, a `view_port` has to be allocated, and callbacks to their functions registered. (The callback functions will later be added to the code)
- `view_port_alloc()` will allocate a new `view_port`. 
- `draw` and `input` callbacks originating from the `view_port` can be registerd with the functions
    - `view_port_draw_callback_set`
    - `view_port_input_callback_set`
- Register the `view_port` to the `GUI`

```c
int32_t hello_world_app(void* p) { 
    osMessageQueueId_t event_queue = osMessageQueueNew(8, sizeof(PluginEvent), NULL); 
 
    // Set system callbacks
    ViewPort* view_port = view_port_alloc(); 
    view_port_draw_callback_set(view_port, render_callback, &state_mutex);
    view_port_input_callback_set(view_port, input_callback, event_queue);
 
    // Open GUI and register view_port
    Gui* gui = furi_record_open("gui"); 
    gui_add_view_port(gui, view_port, GuiLayerFullscreen); 

    ...
}
```

## Callbacks 
Flipper will let the plugin know once it is ready to deal with a new frame or once a button is pressed by the user. 

**input_callback:**
Signals the plugin once a button is pressed. The event is queued in the event_queue. In the main thread the queue read and handled. 

A refrence to the queue is passed during the setup of the application.

```c
typedef enum {
    EventTypeTick,
    EventTypeKey,
} EventType;

typedef struct {
    EventType type;
    InputEvent input;
} PluginEvent;

static void input_callback(InputEvent* input_event, osMessageQueueId_t event_queue) {
    furi_assert(event_queue); 

    PluginEvent event = {.type = EventTypeKey, .input = *input_event};
    osMessageQueuePut(event_queue, &event, 0, osWaitForever);
}
``` 

**render_callback:**
Signals the plugin when flipper is ready to draw a new frame in the canvas. For the hello-world example this will be a simple frame around the outer edges. 

```c
static void render_callback(Canvas* const canvas, void* ctx) {
    canvas_draw_frame(canvas, 0, 0, 128, 64);
}
```

## Main Loop and plugin State
The main loop runs during the lifetime of the plugin. For each loop we try to pop an event from the queue, and handle the queue item such as button input / plugin events. 

For this example we render a new frame, every time the loop is run. This can be done by calling `view_port_update(view_port);`. 

```c
    PluginEvent event; 
    for(bool processing = true; processing;) { 
        osStatus_t event_status = osMessageQueueGet(event_queue, &event, NULL, 100);

        if(event_status == osOK) {
            // press events
            if(event.type == EventTypeKey) {
                if(event.input.type == InputTypePress) {  
                    switch(event.input.key) {
                    case InputKeyUp:   
                    case InputKeyDown:   
                    case InputKeyRight:   
                    case InputKeyLeft:   
                    case InputKeyOk: 
                    case InputKeyBack: 
                        // Exit the plugin
                        processing = false;
                        break;
                    }
                }
            } 
        } else {
            FURI_LOG_D(TAG, "osMessageQueue: event timeout");
            // event timeout
        }

        view_port_update(view_port); 
    }
```

### Plugin State
Because of the callback system, the plugin is being manipulated by different threads. To overcome race conditions we have to create a shared object that is safe to use. 

1. Allocate a new PluginState struct, and initialise it before the main loop.
```c
typedef struct { 
} PluginState; 

// in main:
PluginState* plugin_state = malloc(sizeof(PluginState));
```
2. Using `ValueMutex` we create a mutex for the plugin state called `state_mutex`. 
3. Initalise the mutex for `PluginState` using `init_mutex()`
4. Pass the mutex as argument to `view_port_draw_callback_set()` so we can safely access the shared state from flippers thread. 

```c
typedef struct { 
} PluginState; 

int32_t hello_world_app(void* p) { 
    osMessageQueueId_t event_queue = osMessageQueueNew(8, sizeof(PluginEvent), NULL); 
    
    PluginState* plugin_state = malloc(sizeof(PluginState));
    ValueMutex state_mutex; 
    if (!init_mutex(&state_mutex, plugin_state, sizeof(PluginState))) {
        FURI_LOG_E("Hello_World", "cannot create mutex\r\n");
        free(plugin_state); 
        return 255;
    }

    // Set system callbacks
    ViewPort* view_port = view_port_alloc(); 
    view_port_draw_callback_set(view_port, render_callback, &state_mutex);
    view_port_input_callback_set(view_port, input_callback, event_queue);
...
```


### Main Loop 

Let's deal with the mutex in our main loop. So we can update values from the main loop based on user input. As an example, we will move a hello-world text through the screen. Based on user input. 

1. For this we add a `int x` and `int y` to your state. 
```c
typedef struct { 
    int x; 
    int y;
} PluginState; 
```

2. Initialise the values of the struct using a new `hello_world_state_init()` function. 
```c
static void hello_world_state_init(PluginState* const plugin_state) {
    plugin_state->x = 50; 
    plugin_state->y = 30;
} 
```
Call it after allocating the object in the main function. 
```c
PluginState* plugin_state = malloc(sizeof(PluginState));
hello_world_state_init(plugin_state);
ValueMutex state_mutex; 
...
```
4. Aquire a blocking mutex after a new event is handled in the queue. Write values to the locked `plugin_state` object when user presses buttons. And release when we finish working with the state. 

```c
PluginEvent event; 
for(bool processing = true; processing;) { 
    osStatus_t event_status = osMessageQueueGet(event_queue, &event, NULL, 100);
    PluginState* plugin_state = (PluginState*)acquire_mutex_block(&state_mutex);

    if(event_status == osOK) {
        // press events
        if(event.type == EventTypeKey) {
            if(event.input.type == InputTypePress) {  
                switch(event.input.key) {
                case InputKeyUp: 
                        plugin_state->y--;
                    break; 
                case InputKeyDown: 
                        plugin_state->y++;
                    break; 
                case InputKeyRight: 
                        plugin_state->x++;
                    break; 
                case InputKeyLeft:  
                        plugin_state->x--;
                    break; 
                case InputKeyOk: 
                case InputKeyBack: 
                    processing = false;
                    break;
                }
            }
        } 
    } else {
        FURI_LOG_D(TAG, "osMessageQueue: event timeout");
        // event timeout
    }

    view_port_update(view_port);
    release_mutex(&state_mutex, plugin_state);
}
...
```

## Drawing Graphics 

Creating graphics on flipper has been made easy by flippers developers. An canvas around the outer edges of the screen could be easly added with a single line: `canvas_draw_frame(canvas, 0, 0, 128, 64);`. 

However, when it comes to dealing with user input, moving objects, changing processes we have to take into account that objects might be used by other threads. In the previous part we added a mutex in order to block any other thread writing to an object. For safe drawing graphics, we have to do the same.

1. Aquire a mutex by calling `acquire_mutex()`. The `render_callback()` has the context in a argument. Previously we told the set_callback function to use `plugin_state` for this. 
2. Check if the mutex is valid, otherwise skip this render
3. Do all the drawing that we like.. In this case a simple text Hello world, on the `x` and `y` positions we have set in the plugin_state. 
4. Close the mutex again. 

```c
static void render_callback(Canvas* const canvas, void* ctx) {
    const PluginState* plugin_state = acquire_mutex((ValueMutex*)ctx, 25);
    if(plugin_state == NULL) {
        return;
    }
    // border around the edge of the screen
    canvas_draw_frame(canvas, 0, 0, 128, 64);
    
    canvas_set_font(canvas, FontPrimary);
    canvas_draw_str_aligned(canvas, plugin_state->x, plugin_state->y, AlignRight, AlignBottom, "Hello World");

    release_mutex((ValueMutex*)ctx, plugin_state);
}
``` 

## Closing the plugin

You must clear the resources of the plugin after the loop ends. Let's add the following lines at the end of the main function to ensure that these resources are freed correctly:

```c
PluginEvent evt;
for(bool processing = true; processing;) {
    /* ... */
}

view_port_enabled_set(view_port, false);
gui_remove_view_port(gui, view_port);
furi_record_close("gui");
view_port_free(view_port);
osMessageQueueDelete(event_queue);

return 0;
```

## Building the firmware + plugin

Before the plugin is added to flipper. We have to let the compiler know, where to find the plugins files. To do so, create an `application.fam` file in the plugin directory. This file is used to register the application and define its requirements. Let's add the following configuration:

```c
App(
    appid="hello_world",
    name="Hello World",
    apptype=FlipperAppType.PLUGIN,
    entry_point="hello_world_app",
    cdefines=["APP_HELLO_WORLD"],
    requires=["gui"],
    stack_size=1 * 1024,
    icon="A_Plugins_14",
    order=40,
)
```

Now you can build the application! Follow the instructions from the [official firmware repository](https://github.com/flipperdevices/flipperzero-firmware#build-with-docker), to compile the sources and flash your *Flipper Zero* with your own firmware!
