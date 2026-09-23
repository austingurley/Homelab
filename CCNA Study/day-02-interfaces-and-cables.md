# Day 02 — [Interfaces and Cables]

> Copy this file for each day. Fill from memory after watching, not while watching.
> Video: [x]  Flashcards started: [x]  Lab done: [x]

---

## Concept (why / how it works)

<!-- Explain it like you're teaching someone else. If you can't, rewatch that section. -->
#### **UTP Cables**
- Unshielded twisted pair cables.
- Copper cables that are cheaper than fiber optic connections.
- Used through RJ-45 port on devices and has connectivity options of 10BASE-T, 100BASE-T, 1000BASE-T, and 10GBASE-T.
- UTP cables can only be used for up to ~100m 
#### **Fiber Optic Connections**
- Transmit through light by multimode or singlemode fiber.
- Both are more expensive than UTP, and singlemode is more expensive than multimode.
- Fiber optics connections connect through a SFP Transceiver, Small Form-Factor Pluggable Transceiver. 1GB fiber uses SFP, while 10G fiber uses SFP+.
- Has two strands inside the cable, one sends data one direction and the other cable sends data the opposite direction.
- Singlemode allows longer cables and stronger connection.
- Multimode allows cheaper use while still having higher connection than UTP.
- 1000BASE-LX has multimode 1Gbps (550m length) or singlemode (5km)
- 10GBASE-SR multimode 10Gbps (400m)
- 10GBASE-LR singlemode 10Gbps (10km)
- 10GBASE-ER singlemode 10Gbps (40km)
#### **How they connect**
- Auto-MDIX on a switch means that it is a newer system and the straight through cables are read properly regardless if its Rx->Rx (Receive) or Tx->Tx (Transmit.)
- Straight-through cables connect 1-1, 2-2, 3-3, and 6-6 on 10BASE-T and 100BASE-T UTP cables. This can be used as long as the devices connecting have "opposite ends." Example: A routers Tx is 1 & 2 and Rx is 3 & 6 while a switches Tx is 3 & 6 and Rx is 1 & 2. **(THIS DOES NOT MATTER WITH AUTO-MDIX DEVICES THEY WILL CONFIGURE THEMSELVES EVEN WITH SAME SIDE CONNECTION RX->RX)**
- Straight-through cables for 1000BASE-T and 10GBASE-T UTP cables are the same but use all 8 connectors. Connectors 4, 5, 7, and 8 all push and pull data from end-to-end continuously. 
- Crossover cables allow you to connect from devices that have similar connectors. Routers, clients, servers, and firewalls all have the same Tx and Rx connectors, so you need crossover cables that will instead pair 1-3, and 2-6 both ways. (**Useless again when factoring in Auto-MDIX**)
- **Overall:** straight-through cable for dissimilar devices(router to PC), crossover for similar devices(PC to PC).

**Things I got wrong / confused me:**
- Need to study the types of fiber optics and UTP more. The standards, types, speeds, and distances table in the flashcards.

#### **Charts:**
![](Images/Ethernet-Cable-Standards.png)
![](Images/Fiber-Optic-Cable-Standards.png)

---

## Commands (CLI syntax)

<!-- Only the commands. Copy these into the running cheat sheet too. -->

```
! purpose:
config-command-here
```

```
! verify with:
show ...
```

---

## Lab notes

<!-- What you configured, what broke, how you fixed it. -->

- N/A. Did everything correctly before watching video.

**Screenshot from lab:**
<!-- Screenshot work from todays lab -->
![](Images/ccna-lab-2.png)

---

## Quiz / flashcard flags

<!-- Cards or quiz questions you missed — revisit these before exam week. -->

- Remember Auto MDI-X
- Study charts from above
