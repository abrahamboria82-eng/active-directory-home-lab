# Active Directory Home Lab (Windows Server 2022 + Windows 10 in VirtualBox)

**Abraham Boria** · IT student (AAS in Information Technology) · [LinkedIn](https://linkedin.com/in/abrahamboria) · [GitHub](https://github.com/abrahamboria82-eng)

A hands-on Active Directory lab built to practice the day-to-day work of an IT help desk: networking, DNS, domain joins, user and group management, and troubleshooting. I documented every problem I hit and how I fixed it.

**Demo video:** _add link here after uploading (YouTube unlisted works well)_

## Lab Overview

| Item | Details |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Domain controller | Windows Server 2022 Standard (VM name `DC01`, hostname `Abeiwnlserver-2026`) |
| Client | Windows 10 22H2 (VM name `client01s`) |
| Domain | `lab.local` |
| Network | VirtualBox **Internal Network** named `labnet` (isolated, no internet) |

### IP Addressing

| Machine | IP address | Subnet mask | Gateway | DNS |
|---|---|---|---|---|
| DC01 | 192.168.10.10 | 255.255.255.0 | none | 192.168.10.10 (itself) |
| client01s | 192.168.10.20 | 255.255.255.0 | none | 192.168.10.10 (the DC) |

## Skills Practiced

- Active Directory Domain Services and DNS on Windows Server 2022
- Organizational Units, user accounts, and security groups
- Password resets and account disable/enable
- VirtualBox networking (NAT vs. Internal Network)
- Static IP configuration and DNS troubleshooting
- Command-line diagnostics: `ipconfig /all`, `ipconfig /flushdns`, `ipconfig /registerdns`, `ping`, `nslookup`, `whoami`
- DNS Manager: reading and cleaning up forward lookup zone records
- Joining a Windows 10 client to a domain and signing in as domain users

## Build Steps

### 1. Set up the virtual network

1. Shut down both VMs completely. If a VM shows **Saved**, discard the saved state first, because VirtualBox locks network settings while a VM is running or saved.
2. In each VM go to **Settings > Network > Adapter 1** and set **Attached to: Internal Network**, **Name: `labnet`** (identical on both VMs).
3. Disable **Adapter 2** on both VMs so each machine has only one IP.

### 2. Configure the domain controller (DC01)

1. Press **Win + R**, run `ncpa.cpl`, open the adapter's **Properties > IPv4**.
2. Set IP `192.168.10.10`, mask `255.255.255.0`, no gateway, preferred DNS `192.168.10.10`.
3. Run `ipconfig /flushdns`, `ipconfig /registerdns`, then `nslookup lab.local` and confirm it returns only `192.168.10.10`.
4. Open DNS Manager (`dnsmgmt.msc`) and delete stale records in **Forward Lookup Zones > lab.local** (see the troubleshooting log).

### 3. Configure the client (client01s)

1. Run `ncpa.cpl` and open **Properties > IPv4**.
2. Set IP `192.168.10.20`, mask `255.255.255.0`, no gateway, preferred DNS `192.168.10.10`.
3. Test connectivity: `ping 192.168.10.10` (4 replies, 0% loss) and `nslookup lab.local`.

### 4. Join the client to the domain

1. On the client, press **Win + R**, run `sysdm.cpl`, click **Change**.
2. Select **Domain**, enter `lab.local`, and authenticate with the domain Administrator account.
3. Restart. The sign-in screen now shows **Sign in to: Lab**.
4. On the DC, open **Active Directory Users and Computers** and confirm the client appears under **Computers**.

**Status:** Completed October 2026. The client is joined to `lab.local`, and both domain users below signed in successfully from client01s (verified with `whoami`).

### 5. Organizational Units, users, and groups

| OU | User | AD logon name | Security group |
|---|---|---|---|
| Sales | Jane A Smith | `jsmith12` | SG-Sales |
| IT | Alex Jones | `ajones` | SG-IT |

1. Open **Active Directory Users and Computers** (`dsa.msc`).
2. Right-click `lab.local` > **New > Organizational Unit** (created `Sales` and `IT`).
3. Right-click an OU > **New > User** and set the logon name and a temporary password, with **User must change password at next logon** checked.
4. Right-click the OU > **New > Group** (scope **Global**, type **Security**) and add the user under the **Members** tab.
5. On client01s, choose **Other user** and sign in as `Lab\<logon name>`. Windows forces a password change on the first sign-in.

### 6. Help desk tasks practiced

- **Password reset:** reset a user's password from the DC, required a change at next logon, then signed in from the client to confirm.
- **Disable and enable an account:** disabled a user, confirmed the sign-in failed on the client, then re-enabled the account.

## Troubleshooting Log

### Problem 1: "An Active Directory Domain Controller for the domain lab.local could not be contacted"

The domain name was typed correctly, but the join failed. The cause was networking and DNS, not the name.

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| 1 | Join error even though `lab.local` was spelled correctly | Domain joins depend on DNS finding the DC, and the client was not pointed at the DC for DNS | Put both VMs on the same Internal Network and set the client's DNS to the DC's IP |
| 2 | `lab.local` resolved to `10.0.2.15` and ping to `192.168.10.10` said "destination host unreachable" | DC was still on VirtualBox's default NAT address and had not been given the lab IP | Moved the adapter to `labnet` and set a static IP |
| 3 | Both VMs showed two network adapters on `labnet` | A duplicate adapter gives a machine two IPs, and both get registered in DNS | Disabled Adapter 2 (VM had to be powered off first) |
| 4 | Adapter checkboxes were greyed out in VirtualBox | VM was in a **Saved** state | Discarded the saved state, then edited the settings |
| 5 | `nslookup lab.local` returned a second, wrong address | Stale DNS records from the DC's old name and NAT address (`10.0.2.15`, plus an IPv6 record) | Deleted the stale A and AAAA records in DNS Manager, then ran `ipconfig /flushdns` and `ipconfig /registerdns` |
| 6 | Ping to the DC failed with "reply from 192.168.10.20: destination host unreachable" | The DC had been given the client's IP (`192.168.10.20`), so nothing owned `.10` | Confirmed the machine with `ipconfig /all` (hostname and build number), set the DC to `192.168.10.10` |
| 7 | VM sat at 100% on "Updating your system" | Windows was finishing a pending update after boot | Waited for it to complete instead of force-powering off |

### Problem 2: Active Directory object issues

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| 8 | `SG-Sales` showed as a folder in the OU tree, and had no members option | It was created as an Organizational Unit instead of a Security Group | Turned on **View > Advanced Features**, unchecked **Protect object from accidental deletion** on the Object tab, deleted it, and recreated it with **New > Group** |
| 9 | Domain sign-in failed for `Lab\jsmith` | The account's real logon name was `jsmith12` | Opened the user's Properties > Account tab to read the exact logon name, then signed in as `Lab\jsmith12` |

### Takeaways

- Active Directory depends on DNS. If a join fails, check DNS before anything else.
- Always verify which machine you are on (`hostname`, `ipconfig /all`) before changing settings.
- Test in order: `ping` the IP, then `nslookup` the name, then attempt the join.
- Stale DNS records can cause confusing results, so clean them up after changing a server's name or IP.
- For "I can't log in" issues, check the exact logon name in the user's Account tab before resetting anything.
- In Active Directory Users and Computers, OUs show in the left tree and groups show in the right pane, so the icon and Type column tell you which one you made.

## Screenshots

Rename your screenshot files to match these names and upload them to a `screenshots/` folder in the repo.

| | |
|---|---|
| ![ipconfig on DC01](screenshots/01-ipconfig-dc.png) | ![nslookup on DC01](screenshots/02-nslookup-dc.png) |
| ![Ping from client](screenshots/03-ping-client.png) | ![DNS Manager records](screenshots/04-dns-manager.png) |
| ![Sales OU](screenshots/05-sales-ou.png) | ![IT OU](screenshots/06-it-ou.png) |
| ![SG-Sales members](screenshots/07-sg-sales-members.png) | ![SG-IT members](screenshots/08-sg-it-members.png) |
| ![whoami jsmith12](screenshots/09-whoami-jsmith12.png) | ![whoami ajones](screenshots/10-whoami-ajones.png) |

## Next Steps

- [ ] Add Group Policy (drive mapping, password policy)
- [ ] Create a shared folder with group-based permissions
- [ ] Set up DHCP on the DC
- [ ] Add a NAT adapter for internet access
- [ ] Practice account unlocks and moving users between OUs
- [ ] Add a reverse lookup zone in DNS
