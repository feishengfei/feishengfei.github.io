---
title:  "Statistic on ubuntu18.04"
date:   2018-09-29 17:18:39 +0800
---
Here is the stat.

![ubuntu_usage](https://4bds6hergc-flywheel.netdna-ssl.com/wp-content/uploads/2018/06/ubuntu-usage-infographic-revised.png)

And here is my `ubuntu-report` result.
This is the result of hardware and optional installer/upgrader that we collected:

```json
{
  "Version": "18.04",
  "OEM": {
    "Vendor": "Dell Inc.",
    "Product": "Latitude E5510"
  },
  "BIOS": {
    "Vendor": "Dell Inc.",
    "Version": "A16"
  },
  "CPU": {
    "OpMode": "32-bit, 64-bit",
    "CPUs": "4",
    "Threads": "2",
    "Cores": "2",
    "Sockets": "1",
    "Vendor": "GenuineIntel",
    "Family": "6",
    "Model": "37",
    "Stepping": "5",
    "Name": "Intel(R) Core(TM) i5 CPU       M 560  @ 2.67GHz",
    "Virtualization": "VT-x"
  },
  "Arch": "amd64",
  "GPU": [
    {
      "Vendor": "8086",
      "Model": "0046"
    }
  ],
  "RAM": 8,
  "Disks": [
    750.2
  ],
  "Partitions": [
    712.9
  ],
  "Screens": [
    {
      "Size": "476mmx268mm",
      "Resolution": "1920x1080",
      "Frequency": "59.93"
    }
  ],
  "Autologin": false,
  "LivePatch": false,
  "Session": {
    "DE": "ubuntu-communitheme:ubuntu:GNOME",
    "Name": "ubuntu-communitheme-xorg",
    "Type": "x11"
  },
  "Language": "en_US",
  "Timezone": "Asia/Shanghai",
  "Install": {
    "Media": "Ubuntu 18.04 LTS \"Bionic Beaver\" - Release amd64 (20180426)",
    "Type": "GTK",
    "PartitionMethod": "reuse_partition",
    "DownloadUpdates": true,
    "Language": "en",
    "Minimal": false,
    "RestrictedAddons": false,
    "Stages": {
      "0": "language",
      "1": "language",
      "6": "console_setup",
      "9": "wireless",
      "172": "console_setup",
      "175": "wireless",
      "195": "prepare",
      "207": "partman",
      "229": "start_install",
      "256": "timezone",
      "291": "usersetup",
      "305": "user_done",
      "1482": "done"
    }
  }
}
```
