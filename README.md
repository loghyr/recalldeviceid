This draft describes an extension to the NFSv4.2 protocol.

In particular, it describes the ability to recall all layouts based
on a deviceid via a new LAYOUTRECALL4_DEVICEID case in CB_LAYOUTRECALL.

Discussion of this draft takes place on the NFSv4 working group
mailing list (nfsv4@ietf.org).

## Building

This draft uses the [martinthomson/i-d-template](https://github.com/martinthomson/i-d-template)
build system.  On first run, `make` will clone the template library into `lib/`.

```
make
```

To produce a specific output format:

```
make draft-haynes-nfsv4-recalldevice.txt
make draft-haynes-nfsv4-recalldevice.html
```
