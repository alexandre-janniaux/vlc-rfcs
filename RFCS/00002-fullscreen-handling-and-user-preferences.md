# Fullscreen preference and handling

The current planning for fullscreen user preferences is to have a dedicated
settings in each window provider.

Each window provider can report the current available fullscreen device through
the `vout_window_ReportOutputDevice` function. They are passed through the
window owner callbacks to be made available to the real window user. This is the
first mechanisms to get access to the list of available device for the currently
used window provider implementation.

The other planned way to get the list of available devices is through the
enumeration features of VLC variables. By dynamically listing the available
values for each module-specific variable, the settings can be filled with the
currently available outputs values.

The users can then use the `vout_window_SetFullscreen` function to set the
window in fullscreen state depending on implementation-defined behaviour.

```
/* Set fullscreen on a dedicated screen with module-specific values. */
vout_window_SetFullscreen(window, "\\\\.\\DISPLAY1");
vout_window_SetFullscreen(window, "HDMI-A-1");

/* Set fullscreen depending on implementation-defined behaviour, like using the
 * module-specific variable value. */
vout_window_SetFullscreen(window, NULL);
```

## Current users of the vout_window API

The `vout_window` API is currently used in three different cases:

+ Normal playback for which the user wants the window to follow its user
preferences.
+ Some visualization plugins like glspectrum which are creating their own window
and OpenGL-capable surface, so as to perform their own rendering.
+ The splitter `vout_display` module, which might create multiple windows and
for which the user might expect smart behaviour about where to place the window.

## Issues

While the normal playback use case in fullscreen is completely achieved through
the current planned API, some issues arise for all the other use case.

First, these module-specific variables and implementation-defined behaviour
makes the API hard to use for the splitter. As soon as we would decide to set
all output window to fullscreen without explicitly set a screen, every window
will be fullscreened to the same screen although they could be placed on
different screen following a WM policy, which doesn't look right.

This issue could be workarounded if splitter modules had more information about
the outputs layout and size, which they currently haven't.

The second issue is a standard UI usability issue, which needs arbitration.
Making the default fullscreen display not follow the standard fullscreen policy
of the running window environnement might directly disturb the users.

For example, the expected behaviour is at least that a new playback started in
fullscreen mode should be fullscreened on the user-preference choosed output.
However, it is unclear whether setting the video output in fullscreen while it
was windowed should go to the predefined ouptut or be fullscreened, for
instance, on the main current screen the window is on.

Finally, it is unclear whether visualization should also be fullscreened on the
same device as video ouptut.

All these issues stem from the same fact: the default fullscreen device is a
user preference which is related to UI handling, while `vout_window` modules are
low level wrapper above native or virtual window API.
