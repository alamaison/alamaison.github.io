---
layout: page
title: About
---

This blog contains the random musings of Alexander Lamaison.  Much of
it will be programming-related, with C++ a particular favourite.

Musings are often provoked by work on my various open-source projects.

## Gander: an abstract interpreter for Python written in Java

As part of [my PhD research](/public/thesis.pdf) on advanced type
inference for Python, I developed an
[abtract interpreter](https://github.com/alamaison/gander) capable of
sybolically executing real-world Python programs.  The interpreter is
aimed at type inference but should be applicable to other
program-analysis problems.

## Swish: easy SFTP for Windows Explorer

Windows Explorer has built-in support for FTP but not SFTP.
[Swish](http://www.swish-sftp.org) is an Explorer namespace extension
that add this support. Its main goal is ease-of-use.

## Emacs cmake-project mode

Emacs build tools for C/C++ assume a Makefile exists for the project
but this isn't necessarily the case for projects managed by CMake
which can generate many different types of project file. I developed
[cmake-project.el](https://github.com/alamaison/emacs-cmake-project)
which integrates the CMake build process with the existing Emacs
tools.

## Coliru Javascript library

My first foray into proper Javascript development is
[coliru.js](https://github.com/alamaison/coliru), which allows you to
compile code snippets on-demand directly in your web-page via the
excellent [Coliru](http://coliru.stacked-crooked.com/) online
compiler.
