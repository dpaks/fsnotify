fsnotify is a Go library to provide cross-platform filesystem notifications on
Windows, Linux, macOS, and BSD systems.

fsnotify requires Go 1.16 or newer.

API docs: https://pkg.go.dev/github.com/fsnotify/fsnotify

Platform support:

| Adapter               | OS             | Status                                                       |
| --------------------- | ---------------| -------------------------------------------------------------|
| inotify               | Linux 2.6.32+  | Supported                                                    |
| kqueue                | BSD, macOS     | Supported                                                    |
| ReadDirectoryChangesW | Windows        | Supported                                                    |
| FSEvents              | macOS          | [Planned](https://github.com/fsnotify/fsnotify/issues/11)    |
| FEN                   | Solaris 11     | [In Progress](https://github.com/fsnotify/fsnotify/pull/371) |
| fanotify              | Linux 5.9+     | [Maybe](https://github.com/fsnotify/fsnotify/issues/114)     |
| USN Journals          | Windows        | [Maybe](https://github.com/fsnotify/fsnotify/issues/53)      |
| Polling               | *All*          | [Maybe](https://github.com/fsnotify/fsnotify/issues/9)       |

Linux and macOS should include Android and iOS, but these are currently untested.

Usage
-----
A basic example:

```go
package main

import (
	"log"

	"github.com/fsnotify/fsnotify"
)

func main() {
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		log.Fatal(err)
	}
	defer watcher.Close()

	done := make(chan bool)
	go func() {
		for {
			select {
			case event, ok := <-watcher.Events:
				if !ok {
					return
				}
				log.Println("event:", event)
				if event.Has(fsnotify.Write) {
					log.Println("modified file:", event.Name)
				}
			case err, ok := <-watcher.Errors:
				if !ok {
					return
				}
				log.Println("error:", err)
			}
		}
	}()

	err = watcher.Add("/tmp")
	if err != nil {
		log.Fatal(err)
	}
	<-done
}
```

A slightly more expansive example can be found in [cmd/fsnotify](cmd/fsnotify),
which can be run with:

    # Watch the current directory (not recursive).
    $ go run ./cmd/fsnotify .

FAQ
---
### Will a file still be watched when it's moved to another directory?
No, not unless you are watching the location it was moved to.

### Are all subdirectories watched too?
No, you must add watches for any directory you want to watch (a recursive
watcher is on the roadmap: [#18]).

[#18]: https://github.com/fsnotify/fsnotify/issues/18

### Do I have to watch the Error and Event channels in a separate goroutine?
As of now, yes (you can read both channels in the same goroutine, you don't need
a separate goroutine for both channels; see the example).

### Why don't notifications work with NFS, SMB, FUSE, /proc, or /sys?
fsnotify requires support from underlying OS to work. The current NFS and SMB
protocols does not provide network level support for file notifications, and
neither do the /proc and /sys virtual filesystems.

This could be fixed with a polling watcher ([#9]), but it's not yet implemented.

[#9]: https://github.com/fsnotify/fsnotify/issues/9

Platform-specific notes
-----------------------
### Linux
When a file is removed a REMOVE event won't be emitted until all file
descriptors are closed. It will emit a CHMOD though:

    fp := os.Open("file")
    os.Remove("file")        // CHMOD
    fp.Close()               // REMOVE

Linux: the `fs.inotify.max_user_watches` sysctl variable specifies the upper
limit for the number of watches per user, and `fs.inotify.max_user_instances`
specifies the maximum number of inotify instances per user. Every Watcher you
create is an "instance", and every path you add is a "watch".

These are also exposed in /proc as `/proc/sys/fs/inotify/max_user_watches` and
`/proc/sys/fs/inotify/max_user_instances`

To increase them you can use `sysctl` or write the value to proc file:

    # The default values on Linux 5.18
    sysctl fs.inotify.max_user_watches=124983
    sysctl fs.inotify.max_user_instances=128

To make the changes persist on reboot edit `/etc/sysctl.conf` or
`/usr/lib/sysctl.d/50-default.conf` (some systemd systems):

    fs.inotify.max_user_watches=124983
    fs.inotify.max_user_instances=128

Reaching the limit will result in a "no space left on device" or "too many open
files" error.

### kqueue (macOS, all BSD systems)
kqueue requires opening a file descriptor for every file that's being watched;
so if you're watching a directory with five files then that's six file
descriptors. You will run in to your system's "max open files" limit faster on
these platforms.

The sysctl variables `kern.maxfiles` and `kern.maxfilesperproc` can be used to
control the maximum number of open files.

### macOS
Spotlight indexing on macOS can result in multiple events (see [#15]). A temporary
workaround is to add your folder(s) to the *Spotlight Privacy settings* until we
have a native FSEvents implementation (see [#11]).

[#11]: https://github.com/fsnotify/fsnotify/issues/11
[#15]: https://github.com/fsnotify/fsnotify/issues/15

Related Projects
----------------
- [notify](https://github.com/rjeczalik/notify)
- [fsevents](https://github.com/fsnotify/fsevents)
