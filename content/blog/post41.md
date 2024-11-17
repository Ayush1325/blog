+++
title = "My Week in Code #10"
description = "Updates regarding Linux kernel Devicetree work"
date = "2024-11-17T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux"]
+++

Hello everyone. I was busy with the Linux kernel for most of the week. Let's go over everything.

# Devicetree append properties support

Currently, devictree supports creating and deleting properties but not appending to existing properties. This functionality is vital for complex uses like MikroBUS. I have been working on this for a while and posted [Patch v3](https://lore.kernel.org/all/20241111-append-v3-0-609c09401f3f@beagleboard.org/) last week.

To understand what it allows, let's take an example:

```dts
dts-v1/;

/ {
    node {
        str-prop = "0";
    };
};

/ {
    node {
        /append-property/ str-prop = "1";
    };
};
```

The above devictree will produce the following output:

```dts
dts-v1/;

/ {
    node {
        str-prop = "0", "1";
    };
};
```

# Devicetree overlays path reference support

Devicetree allows references in two contexts:

1. Integer context: In this case the references are resolved to phandle.
```dts
foo = <&bar>;
```

2. String context: In this case, the references are resolved to paths.
```dts
foo = &bar;
```

Runtime overlays only support the former but not the latter. I have created a [patch series](https://lore.kernel.org/all/20241116-overlay-path-v1-0-ac3e121359e9@beagleboard.org/) to fix this asymmetry.

Additionally, `__symbols__` does not support phandles, which makes overlays modifying `__symbols__` somewhat limiting. More context regarding this patch can be found [here](https://lore.kernel.org/devicetree-compiler/44bfc9b3-8282-4cc7-8d9a-7292cac663ef@ti.com/T/#mf0f6ae4db0848f725ec6e2fb625291fa0d4eec71).

# Ending Thoughts

That is all for the week. Hopefully, this series helps keep people updated about my work and attracts potential contributors.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links
- [BeagleBoard](https://www.beagleboard.org/)
- [Devicetree append properties patch](https://lore.kernel.org/all/20241111-append-v3-0-609c09401f3f@beagleboard.org/)
- [Devicetree overlays path reference patch](https://lore.kernel.org/all/20241116-overlay-path-v1-0-ac3e121359e9@beagleboard.org/)
