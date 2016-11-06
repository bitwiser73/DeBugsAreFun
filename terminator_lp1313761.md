# Terminator bug

Terminator crashes when caps lock is toggled after starting editing tab title once terminal get focus back. The callstack shows a reference to gtkentry.c:remove_capslock_feedback (line 10099)

Issue on launchpad: https://bugs.launchpad.net/terminator/+bug/1313761

## Install sources and debug symbols
```bash
mkdir ~/terminator
cd ~/terminator
apt-get source libgtk2.0-0
apt-get install libgtk2.0-0-dbg
```

## Start gdb
```bash
gdb `find ~/terminator/ -type d -printf ' -d %p'` --args python2.7 $(which terminator)
```

## Suspected function
```c
static void
remove_capslock_feedback(GtkEntry *entry)
{
  GtkEntryPrivate *priv = GTK_ENTRY_GET_PRIVATE (entry);

  if (priv->caps_lock_warning_shown)
    {
      gtk_entry_set_icon_from_stock (entry, GTK_ENTRY_ICON_SECONDARY, NULL);
      priv->caps_lock_warning_shown = FALSE;
    }
}
```

## Set a breakpoint
```
(gdb) b gtkentry.c:remove_capslock_feedback
```

## Check variables
```
(gdb) print *entry
```

```
(gdb) print *priv
value has been optimized out
```

```
(gdb) p *((GtkEntryPrivate*)entry)
$8 = {buffer = 0x0, xalign = 0, insert_pos = 0, blink_time = 0, interior_focus = 0, real_changed = 0,
  invisible_char_set = 0, caps_lock_warning = 0, caps_lock_warning_shown = 0, change_count = 0,
  progress_pulse_mode = 0, progress_pulse_way_back = 0, focus_width = 2166272, shadow_type = GTK_SHADOW_NONE,
  progress_fraction = 7,5888483201215469e-320, progress_pulse_fraction = 0, progress_pulse_current = 0, icons = {
    0x21000000a0, 0x300000003}, icon_margin = 1, start_x = 1, start_y = 0, im_module = 0x0}

(gdb) p ((GtkEntryPrivate*)entry)->caps_lock_warning_shown
$11 = 0
```

## Let's crash
```
(gdb) n
10101	  GtkEntryPrivate *priv = GTK_ENTRY_GET_PRIVATE (entry);
(gdb) n
10103	  if (priv->caps_lock_warning_shown)
(gdb) n

Program received signal SIGSEGV, Segmentation fault.
```
The priv->caps_lock_warning_shown is an invalid address.

## Display the callstack
```
(gdb) bt
#0  remove_capslock_feedback (entry=0x108c690) at /build/buildd/gtk+2.0-2.24.23/gtk/gtkentry.c:10103
#1  0x00007ffff59455e7 in ?? () from /usr/lib/x86_64-linux-gnu/libgobject-2.0.so.0
#2  0x00007ffff595e088 in g_signal_emit_valist () from /usr/lib/x86_64-linux-gnu/libgobject-2.0.so.0
#3  0x00007ffff595f212 in g_signal_emit_by_name () from /usr/lib/x86_64-linux-gnu/libgobject-2.0.so.0
#4  0x00007ffff49fa5a8 in _gdk_keymap_state_changed (display=<optimized out>, xevent=<optimized out>)
    at /build/buildd/gtk+2.0-2.24.23/gdk/x11/gdkkeys-x11.c:716
#5  0x00007ffff49f4ab3 in gdk_event_translate (display=display@entry=0xc49020, event=event@entry=0x108a6d0,
    xevent=xevent@entry=0x7fffffffd750, return_exposes=return_exposes@entry=0)
    at /build/buildd/gtk+2.0-2.24.23/gdk/x11/gdkevents-x11.c:2120
#6  0x00007ffff49f5116 in _gdk_events_queue (display=display@entry=0xc49020)
    at /build/buildd/gtk+2.0-2.24.23/gdk/x11/gdkevents-x11.c:2336
#7  0x00007ffff49f51be in gdk_event_dispatch (source=<optimized out>, callback=<optimized out>,
    user_data=<optimized out>) at /build/buildd/gtk+2.0-2.24.23/gdk/x11/gdkevents-x11.c:2397
#8  0x00007ffff6478e04 in g_main_context_dispatch () from /lib/x86_64-linux-gnu/libglib-2.0.so.0
#9  0x00007ffff6479048 in ?? () from /lib/x86_64-linux-gnu/libglib-2.0.so.0
#10 0x00007ffff647930a in g_main_loop_run () from /lib/x86_64-linux-gnu/libglib-2.0.so.0
#11 0x00007ffff4d79447 in IA__gtk_main () at /build/buildd/gtk+2.0-2.24.23/gtk/gtkmain.c:1271
#12 0x00007ffff5426c47 in ?? () from /usr/lib/python2.7/dist-packages/gtk-2.0/gtk/_gtk.so
#13 0x000000000052d432 in PyEval_EvalFrameEx ()
#14 0x000000000055c594 in PyEval_EvalCodeEx ()
#15 0x00000000005b7392 in PyEval_EvalCode ()
#16 0x0000000000469663 in ?? ()
#17 0x00000000004699e3 in PyRun_FileExFlags ()
#18 0x0000000000469f1c in PyRun_SimpleFileExFlags ()
#19 0x000000000046ab81 in Py_Main ()
#20 0x00007ffff7817ec5 in __libc_start_main (main=0x46ac3f <main>, argc=2, argv=0x7fffffffded8, init=<optimized out>,
    fini=<optimized out>, rtld_fini=<optimized out>, stack_end=0x7fffffffdec8) at libc-start.c:287
#21 0x000000000057497e in _start ()
```

## Assembly where it crashed
```
(gdb) x/16i remove_capslock_feedback
   0x7ffff4d11900 <remove_capslock_feedback>:	push   %rbp
   0x7ffff4d11901 <remove_capslock_feedback+1>:	mov    %rdi,%rbp
   0x7ffff4d11904 <remove_capslock_feedback+4>:	push   %rbx
   0x7ffff4d11905 <remove_capslock_feedback+5>:	sub    $0x8,%rsp
   0x7ffff4d11909 <remove_capslock_feedback+9>:	callq  0x7ffff4d09db0 <IA__gtk_entry_get_type>
   0x7ffff4d1190e <remove_capslock_feedback+14>:	mov    %rbp,%rdi
   0x7ffff4d11911 <remove_capslock_feedback+17>:	mov    %rax,%rsi
   0x7ffff4d11914 <remove_capslock_feedback+20>:	callq  0x7ffff4cb5ab0 <g_type_instance_get_private@plt>
=> 0x7ffff4d11919 <remove_capslock_feedback+25>:	testb  $0x10,0x14(%rax)
   0x7ffff4d1191d <remove_capslock_feedback+29>:	mov    %rax,%rbx
```

## rax is NULL
rax should have stored the address of the tab widget

```
(gdb) i r rax
rax            0x0	0
```

**g_type_instance_get_private** returned NULL and the value is not checked, before the segfault we got the following message:

```
/usr/bin/terminator:118: Warning: g_type_instance_get_private: assertion 'instance != NULL && instance->g_class != NULL'
```

## Check widget's reference count with and without provoking segfault (use of capslock)
```
(gdb) p *
$28 = {widget = {object = {parent_instance = {g_type_instance = {g_class = 0xf152b0}, ref_count = 10, ...

(gdb) p ((GtkEntry*)0X108d2f0)->widget->object->parent_instance->ref_count
$30 = 0
```
So ref counter is 0 when it will segfault, that's why address is invalid

## Setup gdb to handle python aswell
1. Install python2.7-dbg

2. Create a gdb pyframe script [**DEPRECATED**]

```
define pyframe
  x/s ((PyStringObject*)f->f_code->co_filename)->ob_sval
  x/s ((PyStringObject*)f->f_code->co_name)->ob_sval
p f->f_lineno
end
```

3. Following https://wiki.python.org/moin/DebuggingWithGdb

```bash
apt-get install python2.7-dbg
echo "add-auto-load-safe-path /usr/bin/python2.7-gdb.py" >> ~/.gdbinit
```

## The python callstack when it won't segfault
```
Breakpoint 1, remove_capslock_feedback (entry=entry@entry=0x108d2d0)
    at /build/buildd/gtk+2.0-2.24.23/gtk/gtkentry.c:10100
Breakpoint 1, remove_capslock_feedback (entry=entry@entry=0x108d2f0)
    at /build/buildd/gtk+2.0-2.24.23/gtk/gtkentry.c:10100

(gdb) py-bt

#24 Frame 0x7fffd73f4d38, for file /usr/share/terminator/terminatorlib/editablelabel.py, line 93, in _on _click_text (self=<EditableLabel(_entry_handler_id=[277L, 278L, 279L, 280L], _entry=<gtk.Entry at remote  0x7fffd73f3e60>, _custom=False, _autotext='user@pc: /home/user 80x24', _label=<gtk.Label at remote 0x7fffd7f4a780>) at remote 0x7fffd7f4a730>, widget=<...>, event=<gtk.gdk.Event at remote 0x7fffd7f1ec
88>, sig=280L)
    self._entry.grab_focus ()
#49 Frame 0x7ffff7e675c0, for file /usr/bin/terminator, line 118, in <module> ()
    gtk.main()
```
Here the **remove_capslock_feedback** function is called because of: /usr/share/terminator/terminatorlib/editablelabel.py, line 93, in **_on _click_text**

## The python callstack before it segfault
```
Breakpoint 1, remove_capslock_feedback (entry=0x108d2f0)
    at /build/buildd/gtk+2.0-2.24.23/gtk/gtkentry.c:10100
(gdb) py-bt
#15 Frame 0x7ffff7e675c0, for file /usr/bin/terminator, line 118, in <module> ()
    gtk.main()
```
**remove_capslock_feedback** doesn't seem do be triggered by a python call, maybe an issue with py-bt ?


## Function **_on_click_text** from /usr/share/terminator/editablelabel.py:93
Let's do more logs with this decorator

```python
log_count = 0
def logger(func):
    def inner(*args, **kwargs):
        global log_count
        print("[{}] {} - {}, {}".format(log_count, func, args, kwargs))
        log_count = log_count + 1
        return func(*args, **kwargs)
    return inner
```

## Edition (double click)
```
[29] <function set_text at 0x7f4898103398> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, 'user@pc: /home/user 80x24'), {}
[30] <function _on_click_text at 0x7f4898103578> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, <EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, <gtk.gdk.Event at 0x7f4898107c38: GDK_BUTTON_PRESS x=294,00, y=13,00, button=1>), {}
[31] <function _on_click_text at 0x7f4898103578> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, <EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, <gtk.gdk.Event at 0x7f4898107c38: GDK_BUTTON_PRESS x=294,00, y=14,00, button=1>), {}
[32] <function _on_click_text at 0x7f4898103578> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, <EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, <gtk.gdk.Event at 0x7f4898107c38: GDK_2BUTTON_PRESS x=294,00, y=14,00, button=1>), {}
```

## Click + focus out
```
[33] <function set_text at 0x7f4898103398> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, 'user@pc: /home/user 80x23'), {}
[34] <function _on_entry_keypress at 0x7f4898103848> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, <gtk.Entry object at 0x7f48980c4e60 (GtkEntry at 0x2bd12f0)>, <gtk.gdk.Event at 0x7f4898107c10: GDK_KEY_PRESS keyval=Return>), {}
[35] <function _on_entry_activated at 0x7f4898103758> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, <gtk.Entry object at 0x7f48980c4e60 (GtkEntry at 0x2bd12f0)>), {}
[36] <function _entry_to_label at 0x7f4898103668> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, None, None), {}
[37] <function set_text at 0x7f4898103398> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, 'user@pc: /home/user 80x23'), {}
[38] <function set_text at 0x7f4898103398> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, 'user@pc: /home/user 80x23'), {}
[39] <function modify_fg at 0x7f4898103a28> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, <enum GTK_STATE_NORMAL of type GtkStateType>, gtk.gdk.Color('#fff')), {}
[40] <function editing at 0x7f48981032a8> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>,), {}
[41] <function editing at 0x7f48981032a8> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>,), {}
[42] <function set_text at 0x7f4898103398> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, 'user@pc: /home/user 80x23'), {}
[43] <function set_text at 0x7f4898103398> - (<EditableLabel object at 0x7f4898133730 (terminatorlib+editablelabel+EditableLabel at 0x2809180)>, 'user@pc: /home/user 80x24'), {}
```
Doesn't help much...

## The EditLabel class

When double clicked the label is replaced by a GtkEntry 'entry' for the edition. When done the label is restored and 'entry' isn't referenced anymore. As a test I kept a reference on entry and Terminator stop crashing.

```python
class EditLabel(gtk.EventBox):
    ...
    def _entry_to_label (self, widget, event):
        """replace gtk.Entry by the gtk.Label"""
        if self._entry and self._entry in self.get_children():
            #disconnect signals to avoid segfault :s
            for sig in self._entry_handler_id:
                if self._entry.handler_is_connected(sig):
                    self._entry.disconnect(sig)
            self._entry_handler_id = []
            self.remove (self._entry)
            self.add (self._label)
            self._entry = None
            self.show_all ()
            self.emit('edit-done')
            return(True)
        return(False)
```

To simplify the debugging I isolated the suspected code into **terminator_gtk_entry.py**

## Let's install more symbols
```bash
apt-get install libglib2.0-0-dbg
```

Callstack now displays more details. A signal 'state-changed' triggers **remove_capslock_feedback** and I can see that one handler of 'entry' is still connected after being destructed.

```
...
#3  0x00007ffff595f212 in g_signal_emit_by_name (instance=0xafd130, detailed_signal=detailed_signal@entry=0x7ffff4a2a91c "state-changed") at /build/buildd/glib2.0-2.40.0/./gobject/gsignal.c:3403
...
```

## What handle connection/disconnection in gtkentry.c ?
1. Find a connect on 'state-changed': **gtk_entry_focus_in**
2. Find the disconnect: **gtk_entry_focus_out**
3. Make crash terminator with a breakpoint on **gtkentry.c:gtk_entry_focus_out**

Segfault without breaking! But **_entry_to_label** is the signal handler of 'focus-out-event' and it's called! Still 'focus-out-event' isn't propagated to GtkEntry. Why ?

## Check GTK documentation
https://developer.gnome.org/gtk3/stable/GtkWidget.html#GtkWidget-focus-out-event

```
>The “focus-out-event” signal:
...
Returns:
TRUE to stop other handlers from being invoked for the event. FALSE to propagate the event further.
...
```

I'm no gtk/pygtk expert but it seems something is wrong in terminator about this. The corrupted GtkEntry isn't aware that focus is out but it needs to be to disconnect its 'state-changed' handler.

## The glorious patch
```bash
diff -Naur /usr/share/terminator/terminatorlib/editablelabel.py editablelabel.py > patch.txt
```

```python
--- /usr/share/terminator/terminatorlib/editablelabel.py	2013-01-30 12:26:17.000000000 +0100
+++ editablelabel.py	2014-09-03 18:46:17.576421411 +0200
@@ -108,7 +108,9 @@
             self._entry = None
             self.show_all ()
             self.emit('edit-done')
-            return(True)
+            # Returning False will ensure that focus out event will be send to
+            # GtkEntry so it will disconnect 'state-changed' handler
+            return(False)
         return(False)

     def _on_entry_activated (self, widget):
```

