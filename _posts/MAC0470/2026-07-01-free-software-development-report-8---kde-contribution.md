---
# https://chirpy.cotes.page/posts/write-a-new-post/
layout: post
title: "Free Software Development Report #8 - KDE Contribution"
date: 2026-07-01 12:49 -0300
categories: [University of São Paulo, KDE]
tags: [git, UI, okular, contribution]
toc: true
comments: false
media_subpath: /assets/img

description: Contributing to KDE's Okular
---

## Introduction

This report will go over my experience contributing to Okular, a universal document viewer and one of KDE's many projects.

Our second assignment in the [FSD2026](https://uspdigital.usp.br/jupiterweb/obterDisciplina?sgldis=MAC0470) course was to be the submission of a contribution to one of many possible FOSS projects. To guide the students in this task, the instructors responsible for the course curated a list of suggestions of projects that we could contribute to. Among them were KDE, Debian, `kw`, `git` and others.

## Contributing to KDE

When deciding what to contribute to, my project duo and I were torn between Debian and KDE, but after watching presentations about each project we ended up choosing **KDE**.

### Setting up the work environment

The work environment setup for KDE development was a lot simpler than it was for the Linux Kernel. Following [this guide](https://develop.kde.org/docs/getting-started/building/kde-builder-setup/) provided by KDE itself I was done with the setup in under an hour.

### Choosing an issue

KDE as a community possesses many a projects, so it is very convenient that KDE has a bugtracking system that spans across all their projects: ["the Bugzilla"](https://bugs.kde.org/). With it, finding an issue that was viable for a first-timer was quite simple - you just need to use the relevant search tag, "junior-job".

It didn't take very long until we came across [this issue](https://bugs.kde.org/show_bug.cgi?id=517754).

The issue outlines a bug in KDE's Okular, the document viewer. It is described as follows:

>STEPS TO REPRODUCE  
>1. open a pdf document and toggle browse button (the cursor changed to open hand)
>2. hold left click
>3. the open hand cursor changed to move cursor
>
>OBSERVED RESULT  
>the cursor changed to move cursor from open hand cursor
>
>EXPECTED RESULT  
>the cursor has to be changed to close hand cursor from open hand cursor
>
>SOFTWARE/OS VERSIONS  
>Operating System: Kubuntu 25.10  
>KDE Plasma Version: 6.4.5  
>KDE Frameworks Version: 6.17.0  
>Qt Version: 6.9.2  

To summarize, holding left click on browse mode changed the cursor shape to the SizeAll (move) cursor shape, instead of the expected Closed Hand (grab) shape.

| | |
|:-:|:-:|
| ![](sizeall-cursor.png) | <img src="cursor-hand-grab.png" width="130" alt=""> |
| Size all cursor | Closed hand cursor |

A maintainer also added:

> We have had this behavior since 2010 but it's right that it's a bit strange.
>
> I think there should not be a long press delay to change to the closed hand.
>
> It should be changed to the closed hand as soon as the cursor is pressed (similar to what gwenview does for example)

This meant a fix would entail two things:

1. Ensure the cursor is changed to the Closed Hand shape when holding left click.
2. Remove the delay between the mouse press and the cursor change.

The issue didn't seem too difficult and it was fairly recent, so that's we went with.
