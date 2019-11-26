# [Wintun Network Adapter](https://www.wintun.net/)
### TUN Device Driver for Windows

This is a layer 3 TUN driver for Windows 2000, XP, and .NET Server 2003. Originally created for 
[WireGuard](https://www.wireguard.com/), it is intended to be useful to a wide variety of projects
that require layer 3 tunneling devices with implementations primarily in userspace.

The original code for NDIS 6.x and 10 (Windows Vista, 7, 8.x, 10+) is located in the `ndis6` branch.

## Installation

[TODO: Switch to a simple shell script using DevCon to install driver]

## Usage

After loading the driver and creating a network interface the typical way using 
[SetupAPI](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/setupapi), 
open the NDIS device object associated with the PnPInstanceId, enabling all forms of file sharing:

```C
TCHAR *InterfaceList = NULL;
for (;;) {
    free(InterfaceList);
    DWORD RequiredBytes;
    if (CM_Get_Device_Interface_List_Size(&RequiredBytes, (LPGUID)&GUID_DEVINTERFACE_NET,
        InstanceId, CM_GET_DEVICE_INTERFACE_LIST_PRESENT) != CR_SUCCESS)
        return FALSE;
    InterfaceList = calloc(sizeof(*InterfaceList), RequiredBytes);
    if (!InterfaceList)
        return FALSE;
    CONFIGRET Ret = CM_Get_Device_Interface_List((LPGUID)&GUID_DEVINTERFACE_NET, InstanceId,
                    InterfaceList, RequiredBytes, CM_GET_DEVICE_INTERFACE_LIST_PRESENT);
    if (Ret == CR_SUCCESS)
        break;
    if (Ret != CR_BUFFER_SMALL) {
        free(InterfaceList);
        return FALSE;
    }
}
HANDLE WintunHandle = CreateFile(InterfaceList, GENERIC_READ | GENERIC_WRITE,
                                 FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE,
                                 NULL, OPEN_EXISTING, 0, NULL);
free(InterfaceList);
...
```

### Ring layout

You must allocate two ring structs, one for receiving and one for sending:

```C
typedef struct _TUN_RING {
    volatile ULONG Head;
    volatile ULONG Tail;
    volatile LONG Alertable;
    UCHAR Data[];
} TUN_RING;
```

- `Head`: Byte offset of the first packet in the ring. Its value must be a multiple of 4 and less than ring capacity.

- `Tail`: Byte offset of the start of free space in the ring. Its value must be multiple of 4 and less than ring capacity.

- `Alertable`: Zero when the consumer is processing packets, non-zero when the consumer has processed all packets 
and is waiting for `TailMoved` event.

- `Data`: The ring data.

In order to determine the size of the `Data` array:

1. Pick a ring capacity ranging from 128kiB to 64MiB bytes. This capacity must be a power of two (e.g. 1MiB). 
The ring can hold up to this much data.

2. Add 0x10000 trailing bytes to the capacity, in order to allow for always-contigious packet segments.

The total ring size memory is then `sizeof(TUN_RING) + capacity + 0x10000`.

Each packet is stored in the ring aligned to `sizeof(ULONG)` as:

```C
typedef struct _TUN_PACKET {
    ULONG Size;
    UCHAR Data[];
} TUN_PACKET;
```

- `Size`: Size of packet (max 0xFFFF).

- `Data`: Layer 3 IPv4 or IPv6 packet.

### Registering rings

In order to register the two `TUN_RING`s, prepare a registration struct as:

```C
typedef struct _TUN_REGISTER_RINGS {
    struct {
        ULONG RingSize;
        TUN_RING *Ring;
        HANDLE TailMoved;
    } Send, Receive;
} TUN_REGISTER_RINGS;
```

- `Send.RingSize`, `Receive.RingSize`: Sizes of the rings (`sizeof(TUN_RING) + capacity + 0x10000`, as above).

- `Send.Ring`, `Receive.Ring`: Pointers to the rings.

- `Send.TailMoved`: A handle to an [`auto-reset event`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createeventa) 
created by the client that Wintun signals after it moves the `Tail` member of the send ring.

- `Receive.TailMoved`: A handle to an [`auto-reset event`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createeventa) 
created by the client that the client will signal when it changes `Receive.Ring->Tail` and `Receive.Ring->Alertable` is non-zero.

With events created, send and receive rings allocated, and registration struct populated, 
[`DeviceIoControl`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-deviceiocontrol)
(`TUN_IOCTL_REGISTER_RINGS`: 0xca6ce5c0) with pointer and size of descriptor struct specified as `lpInBuffer` and `nInBufferSize` 
parameters. You may call `TUN_IOCTL_REGISTER_RINGS` on one handle only.


### Writing to and from rings

Reading packets from the send ring may be done as:

```C
for (;;) {
    TUN_PACKET *Next = PopFromRing(Rings->Send.Ring);
    if (!Next) {
        Rings->Send.Ring->Alertable = TRUE;
        Next = PopFromRing(Rings->Send.Ring);
        if (!Next) {
            WaitForSingleObject(Rings->Send.TailMoved, INFINITE);
            Rings->Send.Ring->Alertable = FALSE;
            continue;
        }
        Rings->Send.Ring->Alertable = FALSE;
        ResetEvent(Rings->Send.TailMoved);
    }
    SendToClientProgram(Next);
}
```

It may be desirable to spin for some time under heavy use before waiting on the `TailMoved` event, in order to reduce latency.

When closing the handle, Wintun will set the `Tail` to 0xFFFFFFFF and set the `TailMoved` event to unblock the waiting user process.

Writing packets to the receive ring may be done as:

```C
for (;;) {
    TUN_PACKET *Next = ReceiveFromClientProgram();
    WriteToRing(Rings->Receive.Ring, Next);
    if (Rings->Receive.Ring->Alertable)
        SetEvent(Rings->Recieve.TailMoved);
}
```

Wintun will abort reading the receive ring on invalid `Head` or `Tail` or on a bogus packet. In this case, Wintun will set the 
`Head` to 0xFFFFFFFF. In order to restart it, reopen the handle and call `TUN_IOCTL_REGISTER_RINGS` again. However, it should 
be entirely possible to avoid feeding Wintun bogus packets and invalid offsets.

## Building

**Do not distribute drivers named "Wintun", as they will most certainly clash with official deployments. Instead distribute 
[the signed MSMs from Wintun.net](https://www.wintun.net/).** 
If you are unable to use MSMs, [consult the MSI creation instructions](msi-example/README.md).

[TBD]

### General requirements

(Note that Microsoft has long since pulled these versions from their own download sites.)

- [Visual C++ 2008 Express](http://the-eye.eu/public/MSDN/Visual%20Studio%202008/en_visual_studio_2008_express_with_service_pack_1_x86_dvd_x15-04728.iso) or higher
- [Windows SDK for Windows 7 and Windows Server 2008 R2, v7.0](http://the-eye.eu/public/MSDN/Windows%207/en_windows_software_development_kit_for_windows_7_and_windows_server_2008_r2_x86_dvd_400018.iso)
- [Windows Driver Kit for Windows Server 2008 SP2, version 6002.18002](http://the-eye.eu/public/MSDN/Windows%20Vista/en_windows_driver_kit_6001.18002_x86_x64_ia64_284919.iso)

`wintun.sln` may be opened in Visual Studio for development and building. Be sure to run `bcdedit /set testsigning on` before to enable unsigned driver loading. The default run sequence (F5) in Visual Studio will build and insert Wintun.

It is recommended to debug using a 32-bit VM, to get around 64-bit driver signing enforcment.

## License

The entire contents of this repository, including all documentation code, is "Copyright © 2018-2019 WireGuard LLC. All Rights Reserved." and is licensed under the [GPLv2](COPYING).
