# CTFs-Solution-Documented-

This repo is aimed to document CTF challenges that i solved.

Right now this entry is from my HTB practice lab screenshots. I am keeping it simple: what i searched, why i searched it, and what the screenshot proves. This is not a polished report, it is more like my own solving notes so i can come back later and remember the path.

All screenshots used here are only from:

`C:\AD,NetworkDocs\Security`

I am not mixing screenshots from the root folder or from the sysadmin lab.

## HTB Practice Lab - Security Log Hunting

This lab was about reading security logs and narrowing down suspicious activity. The main tools in the screenshots are Splunk and Elastic. I was mostly following Sysmon and Windows event logs, then pivoting between IP addresses, process names, command lines, LSASS access, and group change events.

The strongest path i found was:

`randomfile.exe` -> `rundll32.exe` -> `comsvcs.dll MiniDump` -> `lsass.dmp`

Visible IPs, usernames and passwords are from the lab environment.

---

### 1. Start broad with network connections

I started with Sysmon network events and grouped the results by destination IP, source IP, ports and process image. This gave me the first place to look instead of guessing.

The interesting row showed connections involving `10.0.0.91` and source `10.0.0.253`, with processes like `demon.exe`, `randomfile.exe`, `notepad.exe`, and `rundll32.exe`.

![Finding the IP address responsible](screenshots/2026-09-19/security/FindingTheIpAddressResponsible.png)

### 2. Make one table with the useful stuff

After finding the suspicious IP/process area, i used a bigger stats query to compare process activity, ports, count, and timing. This helped make the noisy results easier to read.

The rows that mattered most were still around `10.0.0.91`, especially `rundll32.exe`, `demon.exe`, and `notepad.exe`.

![Advanced table for the investigation](screenshots/2026-09-19/security/AdvancedKqltomapeverythingwefoundinonesingletable.png)

### 3. Try a more strict query

Then i tried combining loaded CLR/network behavior with known images and IP patterns. This one returned zero events.

Still keeping it here because failed searches are part of the process. It shows i tested that idea and moved on.

![Complex query attempt](screenshots/2026-09-19/security/ComplexQueryforbetternarrowingwherecertainEventcodesaresearchedwithipaddresspatternsandimagesalreadyknowen.png)

### 4. Look for injected threads

I searched Sysmon EventCode 8 to find injected thread activity and used avg/stdev to find things above normal. `randomfile.exe` stood out here, which made it worth pivoting from.

![Trying to find injected threads](screenshots/2026-09-19/security/HereTryingtoFindInjectedThreads.png)

### 5. Find what touched rundll32

Next i searched EventCode 8 again, but targeted `rundll32.exe`. This showed the source image as:

`C:\Users\waldo\Downloads\randomfile.exe`

That was a better chain than only saying "rundll32 looks weird".

![File that caused rundll32 activity](screenshots/2026-09-19/security/FoundThefileThatCausedTheSourceImagerundll32.png)

### 6. Check LSASS access

I checked access to `lsass.exe`. The results showed `notepad.exe`, `Sysmon64.exe`, and `rundll32.exe` touching LSASS with high access. `rundll32.exe` and `notepad.exe` were the suspicious ones for this path.

![LSASS dumping found from rundll32](screenshots/2026-09-19/security/LsassDumpingFoundfromrundll32.png)

### 7. Narrow LSASS access more

I filtered EventCode 10 against `lsass.exe` and removed some Microsoft .NET noise from the source image. This kept the suspicious access visible without too many unrelated rows.

![Narrowing down LSASS access](screenshots/2026-09-19/security/NarrowingDownfurtherbyfindingtheSourceImagethatspawenedlsassdumping.png)

### 8. The command line proved the dump

This was the clearest evidence. The command line showed `rundll32.exe` using `comsvcs.dll MiniDump` and writing:

`C:\temp\lsass.dmp`

So the investigation was no longer just "maybe suspicious". There was an actual LSASS dump command.

![rundll32 MiniDump command line](screenshots/2026-09-19/security/FoundtwoProcessNotepadandrundll32.png)

### 9. PsExec command and lab password

I searched for `PsExec` in command lines and found commands using a lab account/password, downloading `comsvcs.dll`, and running against hosts.

I am not typing the password in the README, but it is visible in the screenshot because this is a closed practice lab.

![Password used to dump LSASS through PsExec](screenshots/2026-09-19/security/PasswordUsedToDumpthelsassthroughPS.png)

### 10. Event 4624 XML view

I opened the raw XML for Event 4624. Friendly view is easier to read, but XML view helps when i need exact field names for searching or building a query.

![Event 4624 XML view](screenshots/2026-09-19/security/HereFoundTheXMLofTheEvent4624.png)

### 11. Elastic filtering for group changes

In Elastic, i filtered for admin group membership event codes `4732` and `4733`, then looked at the administrators group. This was checking for account/group changes after the suspicious activity path.

![Elastic filtering and event listing](screenshots/2026-09-19/security/ElasticFilterationAndEventListing.png)

### 12. Add better rows in Elastic

I added rows/fields like user name, member SID, group name, and host name. The first view was not enough, so this made the table easier to read and better for documentation.

![Rows added for better search results](screenshots/2026-09-19/security/RowAddingForBetterSearchResults.png)

## What i learned

- Start broad, then pivot. IP address, process image, target image, command line.
- EventCode 3 is useful for network connections.
- EventCode 8 helped with injected thread/source image pivots.
- EventCode 10 helped with LSASS access.
- Event 4624 XML view is useful when i need exact fields.
- `rundll32.exe` is not automatically bad, but `rundll32.exe comsvcs.dll MiniDump ... lsass.dmp` is a very strong sign.
- Do not ignore failed searches. A zero-result query still tells me something.
