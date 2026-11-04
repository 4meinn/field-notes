# Incoming Calls Don't Ring the Office DECT Phones After a SIP-Trunk Migration: Root-Cause Diagnosis

- **Type:** Infrastructure troubleshooting (VoIP / DECT telephony)
- **Environment:** Small-office FRITZ!Box telephony — two routers meshed, Telekom CompanyFlex SIP trunk, cordless DECT handsets
- **Surface:** SIP trunk registration, DECT base/handset registration, FRITZ!OS telephony
- **Category:** VoIP telephony, network configuration
- **Date:** 2026-11

## Environment & Scope

A small office whose phones are AVM FRITZ!Fon C5 cordless (DECT) handsets, served by two AVM FRITZ!Box 7590 routers joined in a mesh. One box (the "main box") terminates the DSL line and hosts a Telekom CompanyFlex SIP trunk — an Anlagenanschluss / PBX-style trunk with a number block and extensions. The second box is a mesh client reached over the LAN. The telephony had been working, then stopped after the carrier ran a contract renewal and a "system changeover" on the line. Incoming calls no longer rang any office handset.

All identifiers here are anonymized. The vendor and model classes are kept — AVM FRITZ!Box 7590, FRITZ!Fon C5, Telekom CompanyFlex SIP trunk — because the behaviour is specific to them and makes the note useful. IP addresses are placeholders from a private range: main box `192.168.1.1`, second box `192.168.1.3`, admin laptop `192.168.1.99`. The phone numbers are placeholders: a number block with head number `(089) 555000` and two-digit extensions (…11, …13, …16, …23, …24). No real credentials appear — the SIP username and password are never shown and were entered only by the operator. No production or third-party system is exposed.

## Objective

Make incoming calls ring the office handsets again, and understand the actual cause rather than patching around it. The constraint throughout: this is a live business line on someone else's contract, the person who built it was unavailable, and a wrong change could take down the working half of the line. So every step had to be reversible and the working configuration preserved as a fallback.

## Symptoms

Callers got through to "something", but no handset rang. Concretely:

- Incoming calls appeared in the main box's call list (Anrufliste) with the correct caller ID and time. The system was clearly receiving the calls.
- None of the five cordless handsets rang for any of them.
- A test call to a handset's own direct-dial number behaved the same way: the call connected somewhere (ring-back, then an echo-y dead call), but the handset stayed silent.

Two facts shaped the approach:

- A call reaching the Anrufliste only proves it reached *that router*. It says nothing about whether a phone downstream can ring. That distinction turned out to be the whole problem.
- The handsets were powered and had signal — this was not five dead batteries. So the fault sat between "call arrives at the router" and "handset is asked to ring".

## Diagnosis

### Step 1: separate "online" from "can ring"

The reflex reading was "DECT is off" — on the main box the overview did show DECT inactive. But enabling it surfaced the real split instead of fixing anything: the box reported no DECT handsets registered to it at all, while the handsets were plainly working elsewhere. That mismatch — numbers active on a box with no handsets, handsets alive on a box the main one couldn't see — reframed the problem from "a disabled feature" to "the pieces are in two places".

### Step 2: work out which box is which (the 169.254.1.1 trap)

This cost real time and is worth recording. Both routers answer on the FRITZ!Box fallback address `169.254.1.1`, and which one you reach depends on the moment's link, not on the address. Logging in there bounced between the two boxes: one visit showed five registered handsets, the next showed none, with "invalid session" errors in between. Nothing had changed on the boxes — the address was simply ambiguous.

A complication made this worse: the admin laptop had no Ethernet port, and the office Wi-Fi handed it no IPv4 lease (it fell back to a 169.254 APIPA address). The first reliable way into a box was over its IPv6 link-local gateway; the durable fix was to give the laptop a static IPv4 in the telephony subnet and address each box by its real LAN IP.

Lesson, before any change: identify each box by its label / MAC and its real LAN IP, and never trust `169.254.1.1` to mean a specific box.

### Step 3: map numbers against handsets

With the boxes addressed unambiguously, the two halves came apart cleanly:

- **Main box (`192.168.1.1`, DSL):** the CompanyFlex SIP trunk, all numbers registered and green/active. Incoming calls landed in *this* box's call list. Telephony devices on it: only an integrated answering machine and one (now removed) analog port. No DECT handsets.
- **Second box (`192.168.1.3`, mesh client):** all five FRITZ!Fon C5 handsets registered, each with its name and extension assignment intact — but under *Own numbers*, "no numbers configured". The DECT base here was active and the handsets connected.

So the numbers lived on one box and the phones on the other, with nothing bridging them.

### Step 4: confirm it is not a DECT pairing problem

A lot of effort first went into re-registering a handset onto the main box, where the numbers were. It kept failing — partly a classic wrong-box mistake (holding the DECT button on the wrong physical unit while the pairing wizard ran on the other), partly because a DECT handset only pairs cleanly with the base that actually covers it. But even a successful re-pair would have been the wrong fix: the handsets are used near the second box and have their coverage there, and the trunk is a single-registration trunk that can live on only one box at a time. The handsets were not broken and DECT was not the fault — their box just had no line.

## Root cause

The carrier's changeover left the phone system split across the two routers. The SIP trunk was rebuilt on the main box, but the DECT handsets were, and remained, registered on the second box, which came out of the migration with no numbers. Incoming calls therefore reached the main box — which is why they showed up in its call list — but there was no handset there to ring, and the handsets one hop away had no trunk to receive anything. A working phone system had been cut in half and the two halves left on different boxes.

The visible symptom (calls in the log, silent phones) and the reflex hypotheses (DECT disabled, handsets unpaired) both pointed at the endpoints. The actual fault was architectural, and only became obvious once each box was addressed unambiguously and its numbers and devices were listed side by side.

## Fix

Rebuild the SIP trunk on the box that hosts the handsets (the second box), by copying the *known-good* connection settings off the working box rather than re-deriving them:

- Read the live trunk on the main box, field by field: provider type (Telekom CompanyFlex SIP-Trunk, Durchwahl), the area code and base number (Stammnummer), the extension length, the registration number, the SIP username, the outbound proxy, and the options (CLIP-no-screening on, "always register over the internet" on).
- On the second box, create the same trunk with those exact values and add every extension the handsets use (…11, …13, …16, …23, …24, plus the head number as operator extension 0). The SIP password is typed in by the operator from the carrier's export — it is never copied into notes or screenshots.

Saving the trunk on the second box is the switch: because the trunk allows only one registration, the second box registers and the main box's numbers go inactive. Calls now ring the handsets, which already carried their extension assignments. Finally, remove the trunk from the main box so the two boxes don't contend for the same single registration.

Verification, in order: the numbers go green on the second box; a test call from a mobile to an extension rings the right handset; and the main box's copy is removed only after the second box is confirmed working, so the original stays as a fallback until the end.

## Remediation and hardening

- **Back up both boxes before and after.** The incident was only recoverable because the working trunk could be read off the main box field by field. A saved FRITZ!OS config export of each box turns that into a restore instead of a reverse-engineering job.
- **Don't split telephony across boxes.** The box that hosts the DECT base should hold the trunk. If handsets genuinely need coverage in a second location, bridge the boxes properly (IP-client telephony between them) instead of leaving numbers and handsets on separate islands.
- **Treat the trunk config as documented infrastructure.** Record the trunk settings (minus the password) and keep the SIP credentials in a password manager, so the next person isn't rebuilding a live line from a working box while the original installer is on holiday.
- **Remove the IP ambiguity.** Give each box a fixed, documented LAN IP and a physical label stating its role and address; give the admin machine a static IP or a DHCP reservation. Never administer by `169.254.1.1`.
- **Handle the carrier export as a secret.** The CompanyFlex export file and the config screenshots contained the SIP password in clear text. Those are credentials: don't email them around, don't commit them anywhere, and rotate the SIP password if they were exposed.

## Key takeaway

"Everything is online" is not the same as "a phone can ring." The call log proved only that calls reached the router; it said nothing about whether any handset could receive them — and that gap *was* the fault. The problem was not a dead device or a DECT pairing glitch but an architecture left half-migrated: numbers on one box, handsets on another. The fix was to put the two halves back together on the box where the phones physically live, by copying the known-good trunk config rather than guessing, and to keep the working side as a fallback until the new side was proven. When the symptom points at the endpoints but the endpoints check out, step back and look at where the pieces actually live.
