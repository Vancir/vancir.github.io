---
layout: post
title: WinDBG CheatSheet
date: 2022-10-24 10:00:00
tags: ['windbg', 'debug']
---


## Main Extensions 



## Symbols 

* `.sympath`: get/set path for symbol search
* `.sympath +XY`: append XY directory to the searched symbol path
* `!sym noisy`: instructs windbg to display information about its search for symbols
* `dt ntdll!*`: display all variables in ntdll

## PEB and TEB

* `!peb`: display PEB
* `dt nt!_PEB -r @$peb`: full PEB dump
* `!teb`: display TEB

Many WinDbg commands (`lm`, `!dlls`, `!imgreloc`, `!tls`, `!gle`) rely on the data retrieved from PEB and TEB

## Process and Module

* `lm`: list modules
* `lm vm kernel32`: verbose output for kernel32
* `!dlls`: dislay list of modules with loader-specific information
* `!dlls -c kernel32`: only display information of `kernel32`
* `!imgreloc`: display relocation information 
* `!dh kernel32`: display the header for kernel32

# Threads Information 

* `~`: thread status for all threads
* `~0`: thread status for thread 0
* `~.`: thread status for currently active thread
* `~*`: thread status for all threads with some extra info
* `~* k`: call stacks for all threads ~ !uniqstack