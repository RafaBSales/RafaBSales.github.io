---
# https://chirpy.cotes.page/posts/write-a-new-post/
layout: post
title: "Free Software Development Report #7 - Linux Kernel Contribution"
date: 2026-04-21 20:34 -0300
categories: [University of São Paulo, Linux]
tags: [git, email, patches, kernel, mutex]
toc: true
comments: false
media_subpath: /assets/img

description: Contributing to the Linux Kernel
---

## Introduction

This report explores the challenges of making safe refactors in the Linux kernel code and the dynamics of incorporating maintainer feedback.

Our first assignment in the [FSD2026](https://uspdigital.usp.br/jupiterweb/obterDisciplina?sgldis=MAC0470) course was to be the submission of a contribution to the Linux project. To guide the students in this task, the instructors responsible for the course curated a [list of suggestions of patches](https://flusp.ime.usp.br/courses/linux-kernel-patch-suggestions/) that could be used for this assignment.

## Contributing to the Linux Kernel

For this project, [Gustavo C. Arakaki](https://linux.ime.usp.br/~arakaki/) and I decided to contribute by following the first suggestion: **Replacing `mutex_lock(&lock)` and `mutex_unlock(&lock)` calls with `guard(mutex)(&lock)`**.

Since I'd already completed the [Operational Systems](https://uspdigital.usp.br/jupiterweb/obterDisciplina?sgldis=MAC0422&codcur=45070&codhab=504) course the year prior, I had some experience working with mutexes. That familiarity with the tools and the apparent simplicity of the task made it so this seemed like a suitable contribution to undertake.

### The contribution

The Linux kernel often has to deal with concurrency by making sure `read` and `write` operations respect data inconsistency, race conditions and deadlocks.

For that, mutex locks are often utilized, as they guarantee only one thread can access the same data at a time.

The traditional way to do this is to acquire the mutex with `mutex_lock(&lock)` then unlocking it with `mutex_unlock(&lock)`, or

```c
struct mutex lock;
mutex_init(&lock);
mutex_lock(&lock);
// Critical section
mutex_unlock(&lock);
```

This approach can be problematic however, as the developer has to guarantee that `mutex_unlock(&lock)` will **always** be called after `mutex_lock(&lock)`. This can be tricky with nested scopes, as often times you might need to place many different calls to unlock the mutex, or use `goto`, which although common in the Linux kernel, is usually considered bad practice.

Not only that, but it reduces **bug surface**, as future developers refactoring the code don't need to worry about unlocking the mutex.

Therefore, we can use **cleanup guards** defined in `include/linux/cleanup.h` that make sure that the mutex is unlocked **once the scope is exited**.

The code above could then be refactored as

```c
struct mutex lock;
mutex_init(&lock);
guard(mutex)(&lock);
// Critical section
```

or
 
```c
struct mutex lock;
mutex_init(&lock);
scoped_guard(mutex)(&lock) {
    // Critical section
}
```

### The first patch

[Available here.](https://lore.kernel.org/linux-iio/20260415203345.195557-1-rafasales@usp.br/)

To start our assignment, we decided to contribute to the `drivers/iio/light/ltr501.c` driver under the IIO subsystem, which is responsible for the [LTR-501ALS-01](https://esphome.io/components/sensor/ltr501/) light/proximity sensor.

As described above, the patch consisted of replacing the traditional `mutex_lock(&lock)` and `mutex_unlock(&lock)` calls with cleanup guards.

Soon after the patch was sent, we received a reply from [Andy Shevchenko](https://github.com/andy-shev), that gave us some insight on how the patch could be improved.

Most importantly:

- Some of our changes **increased the original locking boundaries** of the mutexes. This was done to simplify the code blocks as there was nothing time consuming outside the original critical section that would leave the lock hanging.  
Upon reconsideration however, this was a **bad decision** as changing locking boundaries is a dangerous operation that needs a better justification than to "ease readability".  
This is because future code refactors could end up increasing the time consumption of the now increased locking boundary, which would leave the lock hanging (deadlock) and lead to performance issues.

- Some return statements could be simplified as follows:  
Instead of
```c
guard(mutex)(&data->lock_als);
ret = regmap_field_write(data->reg_als_rate, i);
return ret;
```  
you can simply return the function call:  
```c
guard(mutex)(&data->lock_als);
return regmap_field_write(data->reg_als_rate, i);
```


- Although this wasn't the original focus of our patch, Andy asked us to sort the include headers of the file according to the [IWYU](https://include-what-you-use.org/) (Include-What-You-Use) principle - which is coincidentally one of the other [patch suggestions](https://flusp.ime.usp.br/courses/linux-kernel-patch-suggestions/).

As such, our next patch would have to be a **patchset** containing a revision of the first patch **and a new commit where the headers are sorted with compliance to IWYU**.

### The second patch

[Available here.](https://lore.kernel.org/linux-iio/20260421021023.563290-1-rafasales@usp.br/)

We addressed the issues brought up by Andy using `scoped_guard()` and added a new commit to the set to sort the include headers according to the IWYU principle.

However, since **IWYU** is not built for the Linux kernel and is not well documented yet, some new problems arose from this new addition to the patch, as pointed out by [Jonathan Cameron](https://github.com/jic23) and [Andy Shevchenko](https://github.com/andy-shev):

Notably:
- Some return statements could be simplified *even further* using `guard(mutex)` with `{}` 
```c
{
  guard(mutex)(&lock);
  return ...;
}
```
instead of relying on `scoped_guard()`, which can't handle return statements inside the scope.

- `<asm/*>` and `<linux/*>` include headers are best kept in separate blocks.

- Some of the include headers pointed out by the **IWYU** tool were too redundant and went unnoticed by us, and could be cut from the commit.

- Some of the include headers can be replaced by other, more appropriate ones (such as `<linux/byteorder/generic.h>` -> `asm/byteorder.h`).

- The `<asm/page.h>` header was needed for `PAGE_SIZE`. However, the code that uses `PAGE_SIZE` was outdated by that point, and as such Jonathan requested that we rewrote that code section using `sysfs_emit_at()` instead of `scnprintf()`, in a new commit.

With this, a new patchset was in order, this time containing **three** different commits:

- One that sorts the include headers according to **IWYU** (adjusting the headers pointed out by the maintainers).

- One that replaces the traditional `mutex_lock(&lock)` and `mutex_unlock(&lock)` calls with cleanup guards, making sure to simplify them according to Jonathan's suggestions.

- One that addresses the outdated code sections by replacing the `scnprintf()` calls with `sysfs_emit_at()`, and removing the `<asm/page.h` include header.

### The third patch

This is a work in progress. Updates coming soon.
