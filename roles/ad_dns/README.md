# ad_dns

A controller's DNS service: where it sends what it doesn't know, and that stale records are removed. Run it on
every controller after promotion. It reads first and changes only what differs; check mode says what it would
change. On a machine without the DNS Server role it fails (in check mode it says what it would set instead).

## Variables

| Variable | Required | Default | What |
|---|---|---|---|
| `ad_dns_forwarders` | yes | | the addresses everything outside the controller's zones goes to, in order (`[192.0.2.53]`) |
| `ad_dns_scavenging_interval` | no | `"7.00:00:00"` | how often stale records are removed, a TimeSpan |

## What it does, and the same by hand

1. **Forwarders**, exactly the list (the promotion often copies the DNS client's servers there):

   ```powershell
   (Get-DnsServerForwarder).IPAddress
   Set-DnsServerForwarder -IPAddress 192.0.2.53
   ```

   Beware a loop: if the forwarder forwards the domain's own zone back to the controllers, a name that doesn't
   exist bounces between them. Forward to a resolver that sends the controllers only their own zone.

2. **Scavenging** on the server, at the interval:

   ```powershell
   Get-DnsServerScavenging
   Set-DnsServerScavenging -ScavengingState $true -ScavengingInterval 7.00:00:00
   ```

3. **Aging** on every directory-integrated primary zone (the domain's and `_msdcs`; not `TrustAnchors`, not the
   automatic ones). Scavenging removes only records in zones with aging on, and only those that registered
   themselves and stopped refreshing (no-refresh and refresh intervals, 7 days each by default); static records
   never age:

   ```powershell
   Get-DnsServerZone | Where-Object IsDsIntegrated | Get-DnsServerZoneAging
   Set-DnsServerZoneAging -Name example.test -Aging $true
   ```
