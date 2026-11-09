# Build guide

Build the whole lab in Cisco Packet Tracer in about 1 to 2 hours. Work top to bottom.

**Lab logins** (same on every device): username `admin`, password `LabAdmin2026`, enable password `LabEnable2026`.

---

## 1. Add the devices

Drag each device in from the bottom-left menu. Then click the device, open the **Config** tab, and set the **Display Name** exactly as shown here.

| Name | Packet Tracer device |
|---|---|
| ISP | Network Devices → Routers → **2911** |
| HQ-R1 | Routers → **2911** |
| BR-R1 | Routers → **2911** |
| HQ-CORE | Switches → **3560-24PS** (multilayer switch) |
| HQ-ACC | Switches → **2960-24TT** |
| BR-SW | Switches → **2960-24TT** |
| SRV1 | End Devices → **Server** |
| INET-SRV | End Devices → **Server** |
| STAFF-PC1, STAFF-PC2, GUEST-PC, BR-PC1 | End Devices → **PC** |

Arrange them like the topology picture in the README. A tidy layout makes your screenshots look good.

## 2. Connect the cables

Choose the cable type, click the first device, then pick the exact port.

| From | Port | To | Port | Cable |
|---|---|---|---|---|
| ISP | G0/0 | HQ-R1 | G0/1 | Copper Cross-Over |
| ISP | G0/1 | BR-R1 | G0/1 | Copper Cross-Over |
| ISP | G0/2 | INET-SRV | FastEthernet0 | Copper Cross-Over |
| HQ-R1 | G0/0 | HQ-CORE | G0/1 | Copper Straight-Through |
| HQ-CORE | G0/2 | HQ-ACC | G0/1 | Copper Cross-Over |
| HQ-CORE | Fa0/10 | SRV1 | FastEthernet0 | Copper Straight-Through |
| HQ-ACC | Fa0/1 | STAFF-PC1 | FastEthernet0 | Copper Straight-Through |
| HQ-ACC | Fa0/2 | STAFF-PC2 | FastEthernet0 | Copper Straight-Through |
| HQ-ACC | Fa0/11 | GUEST-PC | FastEthernet0 | Copper Straight-Through |
| BR-R1 | G0/0 | BR-SW | G0/1 | Copper Straight-Through |
| BR-SW | Fa0/1 | BR-PC1 | FastEthernet0 | Copper Straight-Through |

Router links stay red until step 4 turns their interfaces on. That's normal.

## 3. Turn on the security licence (HQ-R1 and BR-R1 only)

The VPN commands only work after this. On **HQ-R1**, open the **CLI** tab. If it asks *"Would you like to enter the initial configuration dialog?"*, type `no` and press Enter. Then type:

```
enable
configure terminal
license boot module c2900 technology-package securityk9
```

Type `yes` when it asks you to accept, then:

```
end
write memory
reload
```

Press Enter to confirm the reload. Do the same on **BR-R1**.

To check it worked, run `show version`. The table near the bottom should show `securityk9` with `Permanent` or `Evaluation`.

## 4. Paste the configs

For each device, open its file in `configs/`, copy everything, and paste it into the device's **CLI** tab. Answer `no` first if the setup dialog appears. Paste in this order:

1. `ISP.txt`
2. `HQ-R1.txt`
3. `HQ-CORE.txt`
4. `HQ-ACC.txt`
5. `BR-R1.txt`
6. `BR-SW.txt`

Things to know:

- Lines starting with `!` are comments. Read them, because they explain what each part does.
- If a line shows `% Invalid input`, write it down and continue. Send it to me afterwards.
- If the `crypto key generate rsa general-keys modulus 1024` line fails, type `crypto key generate rsa` in config mode and answer `1024`. Then type `ip ssh version 2`.
- After the paste, the router console asks for a username. Use `admin` / `LabAdmin2026`.

## 5. Set up the servers and PCs

These are set in the device windows, not the CLI.

**SRV1** (company server)
- Desktop → IP Configuration → Static: IP `10.10.20.10`, Mask `255.255.255.0`, Gateway `10.10.20.1`, DNS `10.10.20.10`
- Services → DNS → On. Add two A records: `intranet.corp.local` → `10.10.20.10` and `www.example.com` → `203.0.113.10`
- Services → HTTP → On. Edit `index.html` so the page says *Corp Intranet (HQ)*
- Services → SYSLOG → On

**INET-SRV** (pretend Internet website)
- IP `203.0.113.10`, Mask `255.255.255.0`, Gateway `203.0.113.1`, DNS `203.0.113.10`
- Services → DNS → On. Add the A record `www.example.com` → `203.0.113.10`
- Services → HTTP → On. Edit `index.html` so the page says *Welcome to example.com*

**All four PCs:** Desktop → IP Configuration → choose **DHCP**.

## 6. Test it and take screenshots

Save each screenshot into `screenshots/`. The first VPN ping may time out while the tunnel comes up. Just ping again.

| # | Where | What to do | Expected result |
|---|---|---|---|
| T1 | each PC | IP Configuration | STAFF gets 10.10.10.x, GUEST gets 10.10.30.x, BR-PC1 gets 192.168.40.x |
| T2 | STAFF-PC1 | `ping 10.10.20.10`, then browse `http://intranet.corp.local` | Replies, and the intranet page loads (inter-VLAN routing + DNS) |
| T3 | STAFF-PC1, then HQ-R1 | Browse `http://www.example.com`, then on HQ-R1 run `show ip nat translations` | Page loads, and a translation to 198.51.100.2 is listed |
| T4 | GUEST-PC, then HQ-CORE | Browse `http://www.example.com`, then `ping 10.10.20.10`. On HQ-CORE run `show access-lists 130` | Internet works, the ping fails, and the deny line shows matches |
| T5 | BR-PC1, then HQ-R1 | `ping 10.10.20.10` twice and browse `http://intranet.corp.local`. On HQ-R1 run `show crypto isakmp sa` and `show crypto ipsec sa` | Branch reaches HQ through the VPN: state `QM_IDLE` and `#pkts encaps/decaps` going up |
| T6 | Simulation mode | Send a ping from BR-PC1 to SRV1 and click the packet between HQ-R1 and ISP | The packet shows ESP (encrypted) |
| T7 | HQ-R1, HQ-CORE | `show ip ospf neighbor`, then `show ip route` on HQ-CORE | Neighbour 2.2.2.2 is `FULL`, and HQ-CORE has an `O*E2 0.0.0.0/0` default route |
| T8 | STAFF-PC1, then GUEST-PC | Command Prompt: `ssh -l admin 10.10.99.2` | Staff can log in to HQ-ACC; Guest is refused |
| T9 | HQ-ACC | `show port-security interface fa0/1` | `Secure-up`, with a sticky MAC learned |
| T10 | SRV1 | Services → SYSLOG | Log messages from HQ-R1 appear |

## 7. Save and publish

1. In Packet Tracer, use **File → Save As** and save `enterprise-network-lab.pkt` into this folder.
2. In GitHub Desktop, use **File → Add local repository** and pick this folder. When it says it isn't a repository yet, click **create a repository**.
3. Click **Publish repository** and untick "Keep this code private".
4. Add the description: *Enterprise network in Cisco Packet Tracer: VLANs, OSPF, NAT, ACLs and a site-to-site IPsec VPN.*

## If something doesn't work

| Problem | Check |
|---|---|
| A cable stays red | Wrong cable type (step 2), or the interface is shut. Run `show ip interface brief`. |
| PC gets 169.254.x.x | DHCP isn't arriving: on HQ-CORE, `show run interface vlan 10` must show `ip helper-address`, and the trunk must allow the VLAN (`show interfaces trunk`) |
| Staff can't reach the Internet | `show ip route` on HQ-CORE needs the `O*E2` default route (T7); `show ip nat translations` on HQ-R1 |
| Branch can't reach HQ | Ping twice (the first one starts the tunnel); check `show crypto isakmp sa`; make sure step 3 was done on **both** routers |
| SSH refused from Staff | `show ip ssh` should say version 2; redo the RSA key step |
