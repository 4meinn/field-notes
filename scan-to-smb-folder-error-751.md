# Scan to Folder over SMB Fails with Error #751: Root-Cause Diagnosis

- **Type:** Infrastructure troubleshooting (scan to folder / SMB)
- **Environment:** Office multifunction printer (Canon imageRUNNER class) scanning to a Windows workstation share over SMB
- **Surface:** SMB/CIFS, NTFS and share ACLs, Windows network profile and firewall
- **Category:** Windows file sharing, network access control
- **Date:** 2026-10

## Environment & Scope

A color multifunction printer on an office LAN, configured to send scans ("Scan and Send") to a shared folder on a Windows 10/11 workstation over SMB. The device and the workstation sit on the same `/24` subnet.

All identifiers in this writeup are anonymized. Hostnames, user names, and the organization are removed, and IP addresses are placeholders from a private range. The device vendor and model class (Canon imageRUNNER) are kept, because the error code is specific to them and makes the writeup useful to others. Concretely I use workstation `192.168.1.50`, printer `192.168.1.40`, gateway `192.168.1.1`, share name `Scans`, folder `C:\Scans`. No production or third-party system is exposed.

## Objective

The scan destination to the workstation was already set up in the device address book, with host, folder path and credentials entered, but every scan job failed. The goal was to make scans land in the folder reliably, and to understand the actual cause instead of fixing it by trial and error.

## Symptoms

The device printed a send report for each attempt with a "transmission incomplete" status: the destination path was shown, but 0 pages were sent.

Two observations shaped the approach from the start:

- One report line read "the send image does not appear". That is not an error, it only means no scan preview was printed on the report. The line that matters is the error line, which carried the destination path.
- Scan destinations belonging to other users on the same device completed with status OK. Only this one workstation destination returned NG. That immediately narrows the fault to the path to this single PC, not to the device, the scanner, or the network as a whole.

## Diagnosis

### Step 1: read the real error code, not the printed report

The printed report shows a status but no numeric code. The code lives in the job log: Status Monitor, then Send, then Job Log, select the failed job, then Details.

The job reported **#751**: "the destination server may be down, or no network connection was established." Nominally this is a connectivity failure, which is exactly why the first hypotheses were about the network. Keeping that in mind matters later, because the code and the evidence end up disagreeing.

### Step 2: eliminate network reachability

Hypothesis: wrong IP, a different subnet, or Wi-Fi client isolation between the device and the PC. I checked both sides rather than assuming.

On the workstation, `ipconfig` gives the real adapter's IPv4, subnet mask and gateway. Virtual adapters (for example VMware or VirtualBox) advertise their own `192.168.x.x` ranges and are a common way to record the wrong address, so those lines are ignored.

```
IPv4 Address . . . : 192.168.1.50
Subnet Mask  . . . : 255.255.255.0
Default Gateway  . : 192.168.1.1
```

The printer's address-book host matched the PC's real IP, same `/24`, same gateway. I then pinged in both directions: PC to printer succeeded, and the device's own PING tool (Settings/Registration, Preferences, Network, TCP/IP Settings, IPv4 Settings, PING Command) reached the PC.

Both directions succeed, so the network path and client isolation are ruled out. One caveat worth noting: Windows blocks inbound echo requests by default, so the File and Printer Sharing (Echo Request) rule has to be enabled first, or the device-to-PC ping fails for the wrong reason.

### Step 3: eliminate the firewall and network profile

The textbook cause of #751 is the Windows network profile set to Public, which blocks inbound SMB on port 445.

```
Get-NetConnectionProfile
```

The profile was indeed **Public**. I corrected it and opened the sharing rule, from an elevated PowerShell:

```
Set-NetConnectionProfile -InterfaceAlias "Wi-Fi" -NetworkCategory Private
Set-NetFirewallRule -DisplayGroup "File and Printer Sharing" -Enabled True
```

Re-tested the scan: still #751. So the Public profile was a genuine misconfiguration worth fixing, but it was not the cause of this failure. This is the point where the error code stops being a reliable guide.

### Step 4: reproduce the write from an ordinary client

Instead of poking the device again, the decisive move is to reproduce what the device does: connect to the share and write a file, as a normal Windows client.

Opening `\\192.168.1.50\Scans` in Explorer and trying to create a file returned:

```
Access to the destination folder was denied. You need permissions to perform this action.
```

The logged-in user could not write to the share at all. That is the answer. The device was connecting and authenticating every time, and then the server was refusing the write. SMB authorizes a write through two independent layers, the share ACL and the NTFS folder ACL, and the write fails if either one denies it. If a human client cannot write to the share, the scanner never could either.

## Root cause

The destination folder was shared but not writable. Neither the share permission nor the NTFS ACL granted write to the account the scan used, so the device completed the connection and was then rejected at the folder.

The device was never the problem. It reached the workstation and authenticated on every attempt; the rejection happened one layer later. The Public network profile was a separate, latent issue that would have caused a real connection failure later, which is why it was still worth fixing.

Worth calling out: #751 nominally means "cannot reach the server", and that framing sent the investigation down the network path first. The real blocker only surfaced when the write was reproduced directly. When the error code and the reproducible evidence disagree, the evidence wins.

## Fix

Grant write at both layers and re-share if needed, from an elevated PowerShell:

```
icacls C:\Scans /grant "scanuser:(OI)(CI)F"
New-SmbShare -Name Scans -Path C:\Scans -FullAccess "Everyone"
```

If the share already exists, refresh the share permission instead of recreating it:

```
Grant-SmbShareAccess -Name Scans -AccountName "Everyone" -AccessRight Full -Force
```

Verification, in order: create a file in `\\192.168.1.50\Scans` from a client (no "access denied"), then run a scan. The send report returned OK. The corrected Private profile and the firewall rule from Step 3 stay in place.

## Remediation and hardening

The fix above uses Everyone with Full control. It gets scans working, but it is an open, world-writable share and poor hygiene. A hardened setup:

- **Dedicated low-value service account.** Create a limited local account used only for scanning, with a strong password. The printer stores this credential in its address book, so it must not be a real user's account or a privileged one.
- **Least privilege on both layers.** Grant that one account Change/Write on the share, remove Everyone, and set the NTFS ACL to the same account plus administrators. The scanner only needs to drop files in, nothing more.
- **Scope the firewall rule.** Allow File and Printer Sharing inbound only from the printer's IP or the management subnet, not from the entire network.
- **Pin the workstation's IP.** The address-book entry points at an IP. If DHCP reassigns it, the destination silently breaks and the symptom returns as #751. Use a DHCP reservation or a static IP.
- **Keep SMBv1 off.** Confirm the device negotiates SMB2 or SMB3. Re-enabling SMB1 to "make it work" trades a scan problem for a well-known security hole. Verify with `Get-SmbServerConfiguration | Select EnableSMB1Protocol`.

## Key takeaway

The printed report, and even the device's own error code #751 ("server unreachable"), pointed at the network, but the network was fine in both directions. The real cause only appeared once the write was reproduced from an ordinary client, where the share rejected it. Two lessons carry over to any SMB destination problem: a write is authorized by two separate layers, the share ACL and the NTFS ACL, and it fails if either denies it; and when the error code and the reproducible evidence disagree, trust what you can reproduce. Narrowing by "who else already works", then testing connectivity in both directions, turned a vague "it will not send" into a one-line permissions fix.
