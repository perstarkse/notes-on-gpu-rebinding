Details on my dynamic GPU passthrough setup. 

There are a lot of guides to get started with GPU passthrough using driver blacklists, kernel modifiers. This works quite well, but one issue is the generally poor power management of the gpu device (at least on nvidia). 

Seeing the documentation regarding managed="yes" it's clear libvirt has the functionality to manage the pci devices itself. On the backend using "virsh nodedev-detach" and "virsh nodedev-reattach". The issue is x11 binding to the GPU at startup, which makes it hard to change drivers to vfio-pci. Thanks to [u/statelesspencil](https://www.reddit.com/r/VFIO/comments/w3itir/comment/igwph0a/?utm_source=share&utm_medium=web2x&context=3) I found a solution. 

Edit the startup procedure for the display manager to run "virsh detach pci" before x11 can bind to it. 

```
!/bin/bash
virsh nodedev-detach pci_0000_0a_00_0
virsh nodedev-detach pci_0000_0a_00_1
```

This will enable you to login and the GPU will be bound to vfio-pci. However libvirt management wont by itself reset it to the nvidia drivers which have better power management. 

So a solution for me was to create a oneshot service that runs a script "virsh reattach pci" 15 seconds after boot. bind-gpu-to-nvidia being an executable script like the detach script but virsh nodedev-attach pci.

```
[Unit]
Description=Bind second gpu to nvidia drivers at boot, but after x11
After=display-manager.service

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 15
ExecStart=/usr/bin/bind-gpu-to-nvidia

[Install]
WantedBy=multi-user.target
```

´´´
!/bin/bash
virsh nodedev-reattach pci_0000_0a_00_0
virsh nodedev-reattach pci_0000_0a_00_1
```
This enables good power management for the card, easy binding/unbinding by virt-manager.  
