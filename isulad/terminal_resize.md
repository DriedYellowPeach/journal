
When the window size of current terminal changed, foreground process group will get a signal about this event. Some application can chose to re-render current screen, for example, VIM.

```c
struct winsize {
    unsigned short ws_row;
    unsigned short ws_col;
    unsigned short ws_xpixel; /* unused */
    unsigned short ws_ypixel; /* unused */
}
```

Invoke Chain:
```
+---------------+
| isulad client |
+---------------+
                  
                   +------------------------------+
                   | client_console_resize_thread |
                   +------------------------------+

                   +------------------------------+ 
                   |     rest_container_resize    |
                   +------------------------------+

+---------------+
| isulad daemon |
+---------------+
                   +------------------------------+
                   |        rest_resize_cb        |
                   +------------------------------+

                   +------------------------------+ 
                   |     container_resize_cb      |
                   +------------------------------+

                   +------------------------------+ 
                   |  runtime_exec_resize_herlper |
                   +------------------------------+

                   +------------------------------+ 
                   |      rt_exec_resize          |
                   +------------------------------+

+---------------+
|    lcr        |
+---------------+




```

Details:
- client_console_resize_thread, this thread get host terminal windows size at a period of 1 second, this thread will be started after `isula run` or `isula exec`
- the rest request was sent to `isulad` by the client resize thread, the request form is like: `id: String, suffix: String, height: u32, width: u32`.
- the rest request with URL `.../Resize` is routed to `rest_resize_cb`.
- In the isulad daemon, checks on is this daemon still running, is window size argument valid, and call lower runtime resize api.
- rt_exec_resize is a dlsym of `lcr_exec_resize`
