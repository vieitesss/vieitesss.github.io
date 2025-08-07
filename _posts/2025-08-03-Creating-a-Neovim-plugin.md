---
title: Create a Neovim plugin from scratch
date: 2025-08-03 12:23:00 +0200
categories: [Neovim]
tags: [plugin,lua]
---

I want to share with you **how to create your own Neovim plugin** from scratch. We'll go through every step, from creation to testing and releasing the plugin. This is supposed to be a user friendly guide to make it easier to start to anyone who wants to create a Neovim plugin.

# Background

Neovim is configured using the **Lua programming language**. This language is known for its modularity and ability to get integrated in applications written in other languages. In order to follow this guide, you have to be familiarized with this language and how to structure applications with it.

> **I'm not a professional Lua programmer** by a long shot. This is my first, near to complete, Neovim plugin. I have made some scripts or functions, but not something as complex as this. The implementation is not perfect. I'm open to learn. **Any constructive comment is welcome**.
{: .prompt-info }

I'll suppose that you have Neovim installed. This day the version is `v0.11.3`. This post is UNIX/Linux oriented, so, if you are using Windows, you will have to change paths, among other things.

I'll use [lazy.nvim](https://lazy.folke.io/) as the plugin manager. We'll use it to test our plugin without the need of plublishing it in GitHub.

# The structure

L
