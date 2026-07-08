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

KDE as a community possesses many a projects, so it is very convenient that KDE has a bugtracking system that spans across all their projects: ["the Bugzilla"](https://bugs.kde.org/). With it, finding an issue that was viable for a first-timer was quite simple - you just need to use the relevant search tag: "junior-job".

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

To summarize, holding left click on browse mode changed the cursor shape to the Size All (move) cursor shape, instead of the expected Closed Hand (grab) shape.

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

### Working on the issue

The first step to work on the issue is to first build Okular, which can be done using KDE's `kde-builder` tool as follows:

```bash
kde-builder okular
```

This will build Okular locally and install all dependencies.

After doing so, one can run Okular by simply running

```bash
kde-builder --run okular
```

The second step is of course to **reproduce** the bug seen above.

Switching to **Browse mode**, and holding left click, we get the following behavior:

<img src="sizeall.gif" width="600" alt="">

It may be hard to notice from the gif, but there is about a second's delay from when I start holding left click and when the cursor changes to the Size All cursor, which is the behavior described in the bug report.

Having found the bug, we now needed to dive in to the source code. With some help from the [KDE Source Code Cross Reference (LXR)](https://lxr.kde.org/), we were able to find the relevant file responsible for the bug: `/part/pageview.cpp`.

To start working on the file, we created a branch called `fix-cursor-behavior`.

Exploring the code, it didn't take long to understand what caused that behavior:

- Firstly, there was a `QTimer` object called `leftClickTimer`. The `PageView::mousePressEvent()` function, called whenever left click was pressed, started this timer with a delay:
```c++
d->leftClickTimer.start(QApplication::doubleClickInterval() + 10);
```
This means the delay observed was there to "protect" any double-click behavior. The thing is, there was no relevant double-click behavior that could be attributed to Browse mode!

- Secondly, this timer was connected to the `PageView::slotShowSizeAllCursor()` function through the `Qtimer::timeout` signal:
```c++
connect(&d->leftClickTimer, &QTimer::timeout, this, &PageView::slotShowSizeAllCursor);
```
This meant every time the `leftClickTimer` timed out, `PageView::slotShowSizeAllCursor()` was called. As the name implies, that function changes the cursor to the Size All cursor:
```c++
void PageView::slotShowSizeAllCursor()
{
    setCursor(Qt::SizeAllCursor);
}
```

Since the `leftClickTimer` timer and `PageView::slotShowSizeAllCursor()` function were only used for this behavior, removing all mentions of both was a good way to begin making the contribution.

Afterwards, simply adding `setCursor(Qt::ClosedHandCursor);` in `PageView::mousePressEvent()` gave us the behavior we were looking for!

<img src="closedhand.gif" width="600" alt="">

After that, all that was left were treating some edge cases like making sure the cursor changed correctly even when double-clicking or making it so the cursor **didn't** change when hovering over a hyperlink.

### Sending the contribution upstream

Having made all changes we deemed necessary, we now needed to send the contribution for the maintainers to review.

For that purpose, [this tutorial by KDE](https://community.kde.org/Infrastructure/GitLab) was very useful, as it guides you through submitting a merge request to their gitlab instance.

To summarize however, KDE has a gitlab instance where all contributions go through, and contributions are made through **Merge Requests**.

As such, to contribute we needed to create a fork of the repository, push the changes into the fork and submit the merge requests. Then, all we had to do was run some tests in the MR pipeline and the contribution was submitted, pending review.
