---
layout: post
title: "AdaptixC2 windows defender bypass"
date: 2025-09-03
categories:
  - malware
  - cybersec
---

# Disclaimer
This post is for educational and research purposes only. The techniques described are intended to help readers understand how malware and command-and-control systems work, so they can better defend against them. Do not use this information for illegal activities. I take no responsibility for misuse



# Contents
 - [Intro](#intro)
 - [C2 Design & Setup](#c2-design-&-setup)
 - [Installation](#installation)
 - [Overview](#overview)
 - [Custom Loader](#custom-loader)
 - [Capabilities](#capabilities)
 - [Outro](#outro)

# Intro
Hello welcome to another post, today im going to be showing off a relatively new c2 i have been enjoying recently called adaptixc2
Adaptix C2 is a full fledged open source C2 that was created to be an alternative to Cobalt Strike, some of its features include
- Fully Encrypted Communications
- Multiplayer support
- listener plugin extensions
- Socks proxy support
- BOF support
- remote terminal
and more
