# ad_time

A controller's time, by its place in the domain. Time in a domain flows down from the controller that holds the
PDC emulator: it alone follows outside NTP servers; every other controller and every member follows the domain.
Kerberos refuses tickets when clocks differ by more than five minutes, so this matters.

The role reads the machine's role itself (`(Get-CimInstance Win32_ComputerSystem).DomainRole`), so the same role
runs on every controller and puts things right when the PDC emulator moves (run it again after a transfer or
seize). It reads first and changes only what differs; check mode says what it would change. On a machine that is
no controller it fails (in check mode it says it would set the time once it is one).

## Variables

| Variable | Required | Default | What |
|---|---|---|---|
| `ad_time_ntp_servers` | yes | | the NTP servers the PDC emulator follows (`[192.0.2.123]`) |

## What it does, and the same by hand

- **DomainRole 5** (primary: the PDC emulator) follows the list, each asked in client mode (`,0x8`), and announces
  itself a reliable time source (`AnnounceFlags` 5):

  ```powershell
  w32tm /config /manualpeerlist:"192.0.2.123,0x8 192.0.2.124,0x8" /syncfromflags:manual /reliable:yes /update
  w32tm /resync /rediscover
  ```

- **DomainRole 4** (a further controller) follows the domain (`Type NT5DS`, `AnnounceFlags` 10, the defaults):

  ```powershell
  w32tm /config /syncfromflags:domhier /reliable:no /update
  ```

What to check:

```powershell
w32tm /query /source            # the PDC emulator: one of the list; others: a controller
w32tm /query /configuration     # Type: NTP (PDC emulator) or NT5DS
w32tm /query /status            # Leap Indicator 0, Stratum, Last Successful Sync Time
```

The controller needs to reach the NTP servers on UDP 123. A virtual machine whose host also sets its clock (a
hypervisor's time sync) fights this; turn that off for controllers.
