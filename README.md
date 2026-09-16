# Penetration Testing & Lab Write-Ups

A repository of penetration testing walkthroughs and methodology notes across lab environments and retired CTF machines. Write-ups prioritize manual enumeration, service analysis, and command-line verification without relying on automated exploitation frameworks.

## Completed Targets

| Target | Platform | OS | Difficulty | Primary Attack Vector | Privilege Escalation |
| :--- | :--- | :--- | :--- | :--- | :--- |
| [Cap](hackthebox/linux/cap/) | Hack The Box | Linux | Easy | IDOR (PCAP Artifacts) | Linux Capabilities (`cap_setuid`) |

## Repository Structure

Walkthroughs are organized by platform and operating system:
* `hackthebox/<os>/<machine-name>/`
