# rk3328snap
```
Every component are snaped:
gadget snap: idbloader.img, uboot.img, trust.img, system-boot
kernel snap: Image/kernel
core snap: default rootfs (os.snap)
app snap

Ubuntu Core images consist of three snap packages by default. There is a kernel snap, running your hardware, the core snap which contains a minimal root filesystem and the gadget snap which ships a bootloader, the desired partitioning scheme, rules for interface connections and configuration defaults for application snaps included in the image.

Then use 'ubuntu-image' to build an ubuntu core applicance single image

pls see README.md in every components for details

Question:
but how to use custom rootfs ?

```

