---
layout: post
title: "[HTB] Code"
difficulty: Easy
tags: [ssti, path-traversal, sudoers, hashcat]
status: Un-released
updated_date: 2025-07-02
---
***

## Quick Hits:

| Info | Details |
| ---- | ------- |
| **IP Address** | 10.10.11.62 |
| **Difficulty** | 🟢 Easy |
| **Exploits** | Server-Side Template Injection, Path Traversal |
| **Tools Used** | `hashcat` |
| **Techniques Used** | SSH Tunneling, Sudoers |
| **Interesting Files** |  |

#### Port Scan Results:

| Port | State | Service | Version |
| ---- | ----- | ------- | ------- |
| 22/tcp | open | ssh | OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0) |
| 80/tcp | open | http | Apache httpd 2.4.41 ((Ubuntu)) |
