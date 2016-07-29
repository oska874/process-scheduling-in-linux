# Introduction

## About

本文译自 University of Edinburgh 的 Volker Seeker 的 Process Scheduling in Linux ， 介绍了 Linux 的任务调度机制。

## Process Scheduling in Linux

This document contains notes about how the Linux kernel handles process scheduling. They cover
the general scheduler skeleton, scheduling classes, the completely fair scheduling (CFS) algorithm,
soft-real-time scheduling and load balancing for both real time and CFS.
The Linux kernel version looked at in this document is 3.1.10-g05b777c which is used in Android
4.2 Grouper for the Nexus 7.

## Linux 的任务调度

本文是关于 Linux 内核如何进行任务调度的笔记 。 覆盖了通用调度框架 ， 调度类型和完整的公平调度算法（CFS）， 软实时调度以及在 CFS 和实施调度之间进行负载均衡的办法。本文使用的 Linux 内核版本是 3.1.10-g05b777c ， 这个版本同时也是用在 Nexus 7 上的 Android 4.2 Grouper 的内核。

