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


