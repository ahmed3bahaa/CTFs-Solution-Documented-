# CTFs-Solution-Documented-

This repo is aimed to document CTF challenges that i solved.

This is mainly for keeping my HTB practice notes in one place. I add the screenshots, write what i was trying to find, and leave enough notes so i can understand it again later.

## HTB Practice Lab - Security Log Hunting

This one was about hunting through logs and narrowing down suspicious activity. I used Splunk and Elastic, mostly around Sysmon and Windows event logs.

The main chain i ended up with was:

`randomfile.exe` -> `rundll32.exe` -> `comsvcs.dll MiniDump` -> `lsass.dmp`

Anything visible here is from the lab only.

---

### 1. Start broad with network connections

I started with Sysmon network events and grouped the results by destination IP, source IP, ports and process image. This gave me somewhere real to start instead of guessing.

The interesting row showed connections involving `10.0.0.91` and source `10.0.0.253`, with processes like `demon.exe`, `randomfile.exe`, `notepad.exe`, and `rundll32.exe`.

![Finding the IP address responsible](screenshots/2026-09-19/security/FindingTheIpAddressResponsible.png)

### 2. Make one table with the useful stuff

After finding the suspicious IP/process area, i made a bigger table for process activity, ports, count, and timing. The first searches were noisy, so this made it easier to follow.

The rows that mattered most were still around `10.0.0.91`, especially `rundll32.exe`, `demon.exe`, and `notepad.exe`.

![Advanced table for the investigation](screenshots/2026-09-19/security/AdvancedKqltomapeverythingwefoundinonesingletable.png)

### 3. Try a more strict query

Then i tried combining loaded CLR/network behavior with known images and IP patterns. This one returned zero events.

I kept it here anyway because not every query gives a hit, and that is still part of the work.

![Complex query attempt](screenshots/2026-09-19/security/ComplexQueryforbetternarrowingwherecertainEventcodesaresearchedwithipaddresspatternsandimagesalreadyknowen.png)

### 4. Look for injected threads

I searched Sysmon EventCode 8 to find injected thread activity and used avg/stdev to find things above normal. `randomfile.exe` stood out here, which made it worth pivoting from.

![Trying to find injected threads](screenshots/2026-09-19/security/HereTryingtoFindInjectedThreads.png)

### 5. Find what touched rundll32

Next i searched EventCode 8 again, but targeted `rundll32.exe`. This showed the source image as:

`C:\Users\waldo\Downloads\randomfile.exe`

That was better than just saying "rundll32 looks weird".

![File that caused rundll32 activity](screenshots/2026-09-19/security/FoundThefileThatCausedTheSourceImagerundll32.png)

### 6. Check LSASS access

I checked access to `lsass.exe`. The results showed `notepad.exe`, `Sysmon64.exe`, and `rundll32.exe` touching LSASS with high access. `rundll32.exe` and `notepad.exe` were the suspicious ones for this path.

![LSASS dumping found from rundll32](screenshots/2026-09-19/security/LsassDumpingFoundfromrundll32.png)

### 7. Narrow LSASS access more

I filtered EventCode 10 against `lsass.exe` and removed some Microsoft .NET noise from the source image. This kept the suspicious access visible without too many unrelated rows.

![Narrowing down LSASS access](screenshots/2026-09-19/security/NarrowingDownfurtherbyfindingtheSourceImagethatspawenedlsassdumping.png)

### 8. The command line proved the dump

This was the clearest part. The command line showed `rundll32.exe` using `comsvcs.dll MiniDump` and writing:

`C:\temp\lsass.dmp`

So this was not just a suspicious process name anymore. It was actually dumping LSASS.

![rundll32 MiniDump command line](screenshots/2026-09-19/security/FoundtwoProcessNotepadandrundll32.png)

### 9. PsExec command and lab password

I searched for `PsExec` in command lines and found commands using a lab account/password, downloading `comsvcs.dll`, and running against hosts.

The password shown there is just lab stuff, not something real.

![Password used to dump LSASS through PsExec](screenshots/2026-09-19/security/PasswordUsedToDumpthelsassthroughPS.png)

### 10. Event 4624 XML view

I opened the raw XML for Event 4624. Friendly view is easier to read, but XML view helps when i need exact field names for searching or building a query.

![Event 4624 XML view](screenshots/2026-09-19/security/HereFoundTheXMLofTheEvent4624.png)

### 11. Elastic filtering for group changes

In Elastic, i filtered for admin group membership event codes `4732` and `4733`, then looked at the administrators group. This was checking for account/group changes after the suspicious activity path.

![Elastic filtering and event listing](screenshots/2026-09-19/security/ElasticFilterationAndEventListing.png)

### 12. Add better rows in Elastic

I added rows/fields like user name, member SID, group name, and host name. The first view was too plain, so i added what i needed to actually read it.

![Rows added for better search results](screenshots/2026-09-19/security/RowAddingForBetterSearchResults.png)

## What i learned

- Start broad, then pivot. IP address, process image, target image, command line.
- EventCode 3 is useful for network connections.
- EventCode 8 helped with injected thread/source image pivots.
- EventCode 10 helped with LSASS access.
- Event 4624 XML view helps when i need exact fields.
- `rundll32.exe` is not automatically bad, but `rundll32.exe comsvcs.dll MiniDump ... lsass.dmp` is a very strong sign.
- Failed searches are still useful. A zero-result query tells me what path not to waste time on.
