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
