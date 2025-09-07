---
layout: post
title: WSL2 (Linux inside Windows)
date: 2025-08-21 12:00:00 +0000
categories: [Windows, Linux, Development]
tags: [wsl, windows, linux, development]
---

## What is WSL2?

Windows Subsystem for Linux 2 (WSL2) is a powerful feature in Windows that lets you run a Linux environment directly on Windows, without the overhead of a traditional virtual machine or dual-boot setup. It's essentially Linux inside Windows - giving you the best of both worlds.

## Installation Guide

Getting WSL2 up and running is surprisingly simple. Here's how to do it:

### Step 1: Enable Required Windows Features

First, you need to enable the necessary Windows features. Open PowerShell as Administrator and run:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux, VirtualMachinePlatform -All -NoRestart
Restart-Computer
```

This command enables both the Windows Subsystem for Linux and the Virtual Machine Platform features, which are required for WSL2. Your computer will restart after running this command.

### Step 2: Install a Linux Distribution

After your computer restarts, you can install your preferred Linux distribution. Open PowerShell (no need for admin rights this time) and run one of the following commands:

For Debian:
```powershell
wsl --install Debian
```

Or for Ubuntu:

```powershell
wsl --install Ubuntu
```

The system will download and install your chosen distribution. When it's done, you'll be prompted to create a username and password for your Linux environment.

## Getting Started with WSL2

Once installed, you can access your Linux environment by:

1. Opening the Start menu and typing the name of your distribution (e.g., "Ubuntu" or "Debian")
2. Opening Windows Terminal and selecting your distribution from the dropdown

### Useful WSL Commands

Here are some handy commands to manage your WSL environment:

- List installed distributions: `wsl --list`
- Set a distribution as default: `wsl --set-default <DistributionName>`
- Update WSL: `wsl --update`
- Shutdown WSL: `wsl --shutdown`
- Convert from WSL1 to WSL2: `wsl --set-version <DistributionName> 2`

## Tips and Tricks

- Access your Windows files from WSL at `/mnt/c/` (for C: drive)
- Access your WSL files from Windows at `\\wsl$\<DistributionName>\`
- Install Windows Terminal for a better experience when working with WSL
- Use VS Code's Remote - WSL extension to edit code in your Linux environment
- Consider using Docker with WSL2 for containerized development

