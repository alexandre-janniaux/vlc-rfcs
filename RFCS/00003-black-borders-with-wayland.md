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
