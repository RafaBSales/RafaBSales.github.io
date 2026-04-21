---
# https://chirpy.cotes.page/posts/write-a-new-post/
layout: post
title: "Free Software Development - Report #6"
date: 2026-04-21 18:15 -0300
categories: [University of São Paulo, Linux]
tags: [kernel, drivers, C, modules]
toc: true
comments: false
media_subpath: /assets/img

description: The IIO Dummy module
---

This report will follow my experience through the tutorials that can be found at
- [https://flusp.ime.usp.br/iio/iio-dummy-anatomy/](https://flusp.ime.usp.br/iio/iio-dummy-anatomy/), and
- [https://flusp.ime.usp.br/iio/experiment-one-iio-dummy/](https://flusp.ime.usp.br/iio/experiment-one-iio-dummy/).

## The IIO Dummy module

These tutorials focus on the IIO Dummy module - a module that serves as an example of a simple driver in the IIO subsystem. This driver can be found at `drivers/iio/dummy/iio_simple_dummy.c`.

The first tutorial is a study of the structure of the driver, while the second is a more hands-on experiment with it.

## The iio_simple_dummy anatomy

The `iio_dummy` possesses many channels, including accelerometers and voltmeters, which can be found in `iio_dummy_channels[]`, an array of `iio_chan_spec`.

### Channels

Each channel in `iio_dummy_channel[]` contains different fields with different purposes:

- `.type`: defines the category of the measurement (e.g. accelerometers would be `IIO_ACCEL`, voltmeters would be `IIO_VOLTAGE`).
- `.indexed`: a boolean flag. If set to `1` the channel name in the sysfs file will include the number specified in `.channel`.
- `.channel`: the index of the channel.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> As an example, a channel with `.type = IIO_VOLTAGE`, `.indexed = 1,` and `.channel = 0` would be named `in_voltage0_raw`.
{: .prompt-info}
<!-- markdownlint-restore -->

- `.differential`: a boolean flag, set to `1` if the channel measures the difference between two sensors (e.g. `in_voltage1_raw-in_voltage2_raw`).
- `.modified`: a boolean flag. If set to `1`, the value in `.channel2` becomes a modifier - an attribute that identifies the function of the channel's type, like for example a specific axis (e.g. `IIO_MOD_X`, `IIO_MOD_Y` and `IIO_MOD_Z`).
- `.channel2`:
  - If `.differential = 1`, then acts as the negative sensor of the difference.
  - If `.modified = 1`, then identifies the function of the channel's type.
- `info_mask_separate`: defines which attributes are unique to this channel. 

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> `.info_mask_separate = BIT(IIO_CHAN_INFO_RAW)` defines that no other channel will access the raw value obtained by this sensor.
{: .prompt-info}
<!-- markdownlint-restore -->

- `.info_mask_shared_by_type`: defines which attributes are shared by all channels of this type.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> `.type = IIO_VOLTAGE` and `.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE)` defines that all voltage channels use the same scale.
{: .prompt-info}
<!-- markdownlint-restore -->

- `.info_mask_shared_by_dir`: same as above, but refers to the **direction** of the channel (input or output).
- `.scan_index`: the position of this channel's data in the buffer. May be set to `-1` if the channel does not support buffered capture.
- `.scan_type`: defines how the data is formatted in the memory.
- `.output`: a boolean flag. If set to `1` the channel is an output.
- `.event_spec`: a pointer to an array of events.
- `.num_event_specs`: the size of the array above.

### Read and write functions

The dummy module also contains read and write functions. The tutorial uses an outdated version of the dummy module, so the functions look different than the current implementations. Despite that, the main points are still valid:
- Both `iio_dummy_read_raw()` and `iio_dummy_write_raw()` use locking to safely read and write raw data from the channels, utilizing `.info_mask_separate`, `.type`, `.differential` and `.output` to select the correct channel to **read from**/**write to**.

### The probe function

Lastly, we take a look at the `iio_dummy_probe()` function, which
- Allocates memory using `kzalloc(sizeof(*swd), GFP_KERNEL)` and `iio_device_alloc(sizeof(*st))`;
- Initializes device fields (such as setting the device name with `indio_dev->name = kstrdup(name, GFP_KERNEL)`, describing available channels with `indio_dev->channels = iio_dummy_channels`);
- Registers the device with `iio_device_register(indio_dev)`, which exposes the sysfs interfaces, enables user interaction, etc.

## Playing with iio_dummy

Having understood all the tutorials so far, experimenting with this module becomes pretty simple, as it doesn't differ much from what we've seen this far.

We start by compiling, loading and unloading the `iio_dummy` module, which are all operations we have been through in past tutorials.

The differences begin as we mount a `configfs` filesystem somewhere in the VM. It is used to create and configure kernel objects **from userspace**, allowing us to interact with devices by creating/removing directories.

We then proceed to create an entry for one of these devices **by creating a directory** under our mountpoint, reading from the attributes under `/sys/bus/iio/devices/*`, then removing it **by removing the directory**.

The rest of the tutorial walks us through making changes to the source code of the dummy driver and testing the changes, which would be repetitive to include in this report given we already went through the source code.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> It is however worth pointing out that since the tutorial contains an outdated version of the dummy module, the changes to the `iio_dummy_read_raw()` must actually be done to the `__iio_dummy_read_raw()` function, since the former now calls it. Some other changes like adjusting the return types may also be needed.
{: .prompt-info}
<!-- markdownlint-restore -->

## Conclusion

The study of the anatomy of this module introduces many concepts that were very new to me so, as with [Report #4](/posts/free-software-development-report-4/), it took some time for me to piece it all together. That said, going through it again, experimenting with the dummy module *and especially writing this report* certainly helped me get a good grasp of the basics of IIO channels.

