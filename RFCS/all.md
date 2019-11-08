# Smooth resizing -- wayland support

// TODO

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

# Usage of the window in the playback pipeline

// TODO

# Black borders handling

Black borders during playback are handled by the `vout_display` module. However,
even if they are easy to display with a rendering API like OpenGL or Vulkan,
they somehow conflict with how some windowing API are handling buffers. Wayland
windowing is one of these case with maybe some X11 protocols.

Currently Wayland video output doesn't have black borders. Attempts have been
made to support them but there are multiple issues related to this.

First, in Wayland, the size of a surface consists of the size of its content, so
to draw black borders, you either have to write black borders in the content
itself, making a copy of the picture, or shift the surface and add additional
subsurfaces with black content.

But:

+ Copying pictures is very expensive.
+ With some case involving hardware buffer, you cannot copy the pictures.
+ To offset the `wl_surface`, you need this `wl_surface` to have a role of
`wl_subsurface` and have access to the `wl_subsurface` to set its relative
position. In the Wayland model, the `wl_subsurface` state is actually linked to
the state of the parent and thus should be handled in the `vout_window` module
instead of the `vout_display` one, or the subsurface should be created in the
`vout_display`.

Then, the last point also raise differents issues:

When the `vout_display` created the `wl_subsurface` for the video, it should
still fill the given parent `wl_surface` with contents, which in this case can
be nothing other than the background itself. So it should be filled with the
black content below the video. But then it means that the surface layout in case
of embedding is:

+ the black SHM surface
+ the video surface, which might use hardware buffers
+ some SHM or OpenGL surface for subtitles, which might not blend into the video
surface.
+ the interface, which is probably OpenGL rendering in the case of QML for now.

It raises other issues in the case of STB use cases, because the underlying
Wayland compositor might not be able to correctly composite as underlay and
might only be able to use overlays. Then this layout prevents the correct usage
of hardware planes and overlay and might results in compositing bugs, like the
video surface being removed because no composition can be made from it.

Instead, the black borders should either be handled by the interface itself
through correct and synchronous signaling, or at least be handled at the same
level and surface type as subtitles. In this case, the `vout_display` has to
report the area it is able to use for rendering and expect that the underlying
window module offsets the surface correctly and add black borders if needed with
the method of its choice.

In any case, the offset coming from the window module is needed as in case of
embedded window, Client Side Decoration might be used.

Tracking issues:

+ #18045: Wayland (SHM) window black borders missing 
+ #18434:  Paint black bars in OpenGL

# Splitter

The video splitter is currently a `vout_display` with a `dummy` `vout_window`.

It loads a `video_splitter` modules which defines the different windows to open.

Because of this architecture, the splitter doesn't support multi-track inputs
and has to handle the synchronization between each display itself.

Other issues



Use cases:

+ HMD splitter output, so as to use direct display or
`vout_window_SetFullscreen` on the correct output in extended mode.

Current issues:

+ #4106: wall/panoramix: different names for each windows
+ #22674: Display is opened at a different size than window
+ VSYNC-capable output are run synchronously, so VSYNC are cumulated.
+ HW-accelerated decoding doesn't work with splitter, because of `picture_Copy`.


# Window provisioning

## Abstract

In the 3.0 paradigm and more or less the current 4.0 implementation, the window
for playback is provided by a window provider module with the capability `vout
window` according to `vlc_module_load` rules.

It has a few drawbacks:

+ the video window and the interface are usually tangled together when it comes
  to embedding the video into the interface, but the only current ways for the
  interface to share a state with vout windows are through globals or VLC
  variables hacks.

+ the interface cannot really control which vout_window is used for which
  output as everything is done beforehand.

+ it doesn't play well with HMD displays which might need a different window
  provider for direct mode or non-embedded playback.

+ it also doesn't play very well with mosaic use cases.

In addition, we were maintaining hacks inside the Qt interface so as to provide
native surface from Qt Widgets with the correct properties, and such code had to
be duplicated and adapted for any new interface or window integration.

Instead, we would like to have a way to inject windows directly inside playback
pipeline from the interface or another window provider source, the default being
the previous module-based window provider selection.

## Current architecture

Currently, the `vout window` object is created by `vout_display_window_New` from
a module name and a video output. 

@startuml

title Creation of the vout_window and vout_display

vlc_player -> input_resource:
input_resource -> vout_thread: << vout_Create >>
vout_thread -> vout_window: << vout_display_window_New >>
vout_window -> vout_thread: The window has been created
vout_thread -> vout_display: << vout_Start >>
@enduml

The `vout_thread` is also recycled between each playback with the `vout_window`
so as to be able to configure the `vout_window` before usage. The `vout_window`
lifetime is shorter than the `vout_thread`.

A `vout_window` can be `enabled` or `disabled`. The native resources it carries
are expected to be made available in the `enabled` state, but might not be
available in the `disabled` state.

With the push model and `vlc_decoder_device` changes, the `vout_window`
lifecycle becomes tied to the decoder with the constraint that the window must
be `enabled` at least longer than the `vlc_decoder_device`, on which the
`decoder` might depends upon.

## Expected changes

The `vout_window` might be used by other use case than `vout_thread`. The
current obvious other use case is through the `video splitter` output which
creates windows through `vout_window_New` without tying the new created windows
to a `vout_thread`.

To achieve this while keeping clear lifetimes, the `vout_window` must be made
available outside of the `vout_thread` and the `vout_thread` must be killable.

There are orthogonal questions about whether the `vout_thread` should be kept
across playbacks with regards to filters then, but they are not exposed in this
document.

In addition, we would like the window to be provided by the interface somehow.
As the link between the interface and the video pipeline goes through the
`vlc_player` object, it becomes the first-class candidate for bringing the
window down to the video pipeline.

## Suggestion

The following window model is a suggestion made to fix [#22156] while taking
into account the relevant issues related to splitter, multiple output pipeline,
new video pipeline with push+decoder device and native future mosaic use case
like they were mentionned in the 2019 VideoLAN video workshops.

[#22156] Qt Redesign: add wayland window integration
         https://trac.videolan.org/vlc/ticket/22156

Because of the situation including multiple window clients, like multiple video
outputs, we can't just have a `vlc_player_SetWindow(vout_window_t *window)` API
function. In addition, it raises issues when it comes to setting the owner part
to do the same work as `vout_display_window_New` and link the window change to
changes in the `vout_thread` and `vout_display`.

Instead, the system relies on a provider interface, with a callback to get the
window, and a fallback system to use the former module system if the callback
fails.

The interface trying to support window embedding then enable this mechanism by
implementing this `vlc_window_provider` interface and signalling it through
`vlc_player_SetWindowProvider` or maybe `vlc_intf_SetWindowProvider` depending
on how we want the mechanism to be exposed to control interface and other
interface too. The interface mainly consists of a function to initialize a
window from the provider, much alike the `Open` callback of `vout_window`
modules, and a function to return the `vout_window` to the provider after its
usage. The callback to initialize also gets a `vlc_window_client_handle` which
allow the application to signal that the `vout_window` client that the interface
should now be returned through the other function so that the underlying
resources can be reused for a different use case.

Then, the interfaces (especially Qt and LibVLC here) need a common ground to
implement the different window embedding use cases. The planned system is quite
close to the drawable window modules, which allowed the implementation of
function like `libvlc_media_player_set_xwindow`, except that they are
owned-variant of this system. It means that the surface itself is the property
of the "subsurface" window module instead of being external.

To make this work, a new capability is introduced, `vout_subwindow`, which is a
`vout_window`-like module created from another `vout_window` instance.

The first ideas were to have the `vout_subwindow` directly be created as
`vout_window` instances, but it is finally easier to have a wrapping
`vout_window` module controlling the `vout_subwindow` and giving feedback from
the video pipeline to the UI.

```
/* in the Qt interface, pseudo code */
int GetWindow(vlc_window_provider* provider,
              vout_window *window,
              vlc_window_client_handle *handle)
{
    /* We define the host surface for the subwindow. */
    vout_window_handle handle = {
        .display.wl = wayland_display_from_UI,
        .handle.wl  = wayland_surface_from_UI,
    };

    static const vlc_subwindow_callbacks subsurface_cbs = {
        ...
    };

    /* The subwindow can be moved and resized through a dedicated API, without
     * conflicting with limits from the vout_window interface. */
    vlc_subwindow *subwindow =
        vlc_subwindow_New(handle, owner, &subssurface_cbs);

    static const vlc_subwindow_wrapper_callabcks wrapper_cbs = {
        ...
    };

    /* The wrapping vout_window allow the interface to get events like
     * fullscreen or window state from the video pipeline. */
    vlc_vout_window_WrapSubwindow(window, subwindow, owner, &wrapper_cbs);

    return VLC_SUCCESS;
}
```

As for LibVLC, it means that integration into an hosting application is done in
two different ways:

+ either the application doesn't want to know the details and do embedding, and
then no subsurface can be created. It will create a native window like when not
setting anything on the media player currently.

+ or the application want the video to integrate into an existing interface, and
LibVLC will be able to refactor the integration code like `qt/interface_widgets`
into the subwindow mechanism, and provide a way for the interface to control the
integration itself. It will allow implementing Wayland embedding in a much more
natural way and remove the concept of drawable which was considered a bit dirty.


