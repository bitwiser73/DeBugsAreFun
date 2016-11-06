# gnome-settings-daemon (patched)

Easy one: with "**acpi_backlight=vendor**" configured in grub I met some segfault when using nVidia drivers. It crashes after waiting some time with no input.

## Rebuild gnome-settings-daemon with symbols
Has there's no package gnome-settings-daemon-dbg it's needed to rebuild with symbols

```bash
apt-get build-dep gnome-settings-daemon

DEB_BUILD_OPTIONS="nostrip noopt" fakeroot apt-get -b source gnome-settings-daemon/

dpkg -i gnome-settings-daemon_3.2.2-0ubuntu2.1_amd64.deb
```

## Either run gdb or attach
### Run
```bash
gdb `find ~user/gnome-settings-daemon-3.2.2/ -type d -printf ' -d %p'` gnome-settings-daemon
```

## Attach sur gnome-settings-daemon
### Local
```bash
gdb -d ~user/gnome-settings-daemon-3.2.2/plugins/power/ gnome-settings-daemon $(pidof gnome-settings-daemon)
```

## Crash
Wait with no input so it'll crash when trying to handle backlight

```
(gdb) C
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x00007f8cbf5f1C26 in idle_set_mode (manager=0x1C48010,
    mode=GSD_POWER_IDLE_MODE_DIM) at gsd-power-manager.C:2675
2675	                        g_warning ("failed to get existing backlight: %s",
(gdb)
```

## BackTrace

```
(gdb) bt full
#0  0x00007f8cbf5f1C26 in idle_set_mode (manager=0x1C48010,
    mode=GSD_POWER_IDLE_MODE_DIM) at gsd-power-manager.C:2675
        ret = 1
        error = 0x0
        idle = 0
        idle_percentage = 0
        min = -1342016112
        max = 32652
        now = -13
        action_type = GSD_POWER_ACTION_BLANK
        state = GNOME_SETTINGS_SESSION_STATE_ACTIVE
#1  0x00007f8cbf5f24da in idle_evaluate (manager=0x1C48010)
    at gsd-power-manager.C:2918
        is_idle_inhibited = 0
        timeout_blank = 3221508608
        timeout_sleep = 32652
        on_battery = 0
#2  0x00007f8cbf5f309b in idle_idletime_alarm_expired_cb (idletime=0x1C0af80,
    alarm_id=1, manager=0x1C48010) at gsd-power-manager.C:3249
No locals.
#3  0x00007f8cce4640a4 in g_closure_invoke ()
   from /usr/lib/x86_64-linux-gnu/libgobject-2.0.so.0
No symbol table info available.
```

## Crash suspect: gsd-power-manager.c
```
(gdb) l
2670	                        return;
2671	                }
2672
2673	                now = backlight_get_abs (manager, &error);
2674	                if (now < 0) {
2675	                        g_warning ("failed to get existing backlight: %s",
2676	                                   error->message);
2677	                        g_error_free (error);
2678	                        return;
2679	                }
```

## Explanation
The 'error' pointer is NULL initialized so when used in error->message (line 2675)... The **backlight_get_abs** function returns -13 and should fill the error struct.

```
2673	                now = backlight_get_abs (manager, &error);
```

