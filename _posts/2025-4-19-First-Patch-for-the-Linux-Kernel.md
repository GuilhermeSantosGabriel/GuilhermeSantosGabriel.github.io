---
layout: post
title:  "First patch for the Linux Kernel: deduplicating code in the amdgpu driver"
---
Guilherme Santos Gabriel

**This is the first** of the two patches made in tandem with Gabriel Dimant and Andre Jun Hirata for the MAC0470 course on Free Software Development. Here we will be (atempting) to create a patch that tries to solve a **duplicated piece of code** inside the DRM subsystem (drivers/gpu/drm/amd/amdgpu/) of the Linux Kernel.

## The Subsystem

The subsystem that we chose was taken from a list made by the MAC0470 team of possible points of improvement. The file that we ended up tackling was part of the **amdgpu** driver, the driver from the drm subsystem that supports mostly (if not all) AMD Radeon GPUs (thank you amdgpu, my notebook depends on you!).

## What was identified?

As the amdgpu driver is responsible for a myriad of different GPUs, it is fairly common to have different versions of files that are slightly different so as to accomodate both new and old graphics cards. Sometimes implementations are not actually changed: as one version is copied and then pasted to a new file dedicated to new versions, some functions actually end up beeing exactly the same but with a fresh coat of paint (AKA a new function name).

That was the scenario that sumarized the files gmc_v10_0.c and gmc_v11_0.c of the driver: Both of these files represent different versions of the same functionality and ended up with two **100% identical functions:**

* *gmc_v10_0_get_vm_pde | gmc_v11_0_get_vm_pde*

* *gmc_v10_0_get_vm_pte | gmc_v11_0_get_vm_pte*

## What we did at first

These functions did the exact same thing in the grand scheme of things, probably because of the aforementioned "copy and paste" method. We thought that it was a good enough reason to regroup these functions as a single implementation. At first, the solution sounded simple: **We could have the newer file simply have the functions as a "wrapper" of the implementation located on the other files.** This solved the duplication problem and allowed us to reduce the code. 

But this had a problem. **Imagine you are a developer and you found a bug in the *gmc_v10_0_get_vm_pde* function** as a fellow member of the open source community you take it into your own hands and fix the bug with a slight modification to the function. You feel really happy for your accomplishment, as the newer *gmc_v**11**_0_get_vm_pde* function stops working and your gpu catches fire. Okay, maybe not that last part.

That happened was that simply having *gmc_v11_0_get_vm_pde* wrap around *gmc_v10_0_get_vm_pde* makes it so that you silently make the *v11_0* version **dependant** on *v10_0*. We needed a new way.

## What we ended up doing

The solution we ended up with was using a **helper file**. It simply had the functions we needed (alongside a couple of changes caused by macro name conflicts) moved to the file *amdgpu_gmc.c*. We grouped the function to *amdgpu_gmc_nv_get_vm_pde* and *amdgpu_gmc_nv_get_vm_pte*, using "nv" to show they were not version dependant.

## Sending the patch

After these changes we started working on the patch. I was listed as co-developed-by, as my buddy Jun sent the patch itself. 

### THE PATCH MESSAGE:
```
drm/amdgpu: unify gmc v10 and v11 get_vm_pde and get_vm_pte into common helpers

gmc_v10_0_get_vm_pde, gmc_v10_0_get_vm_pte and their v11 counterparts
are identical. Move the shared implementation to amdgpu_gmc.c as
amdgpu_gmc_nv_get_vm_pde and amdgpu_gmc_nv_get_vm_pte, and update both
gmc_v10_0 and gmc_v11_0 to use the common helpers.

No functional changes intended. BUG_ON preserved from original
gmc_v10_0 and gmc_v11_0 implementations.

Signed-off-by: Andre Jun Hirata <andrejhirata@usp.br>
Co-developed-by: Gabriel Dimant <gabriel.dimant@usp.br>
Signed-off-by: Gabriel Dimant <gabriel.dimant@usp.br>
Co-developed-by: Guilherme Gabriel <guilhermesangabriel@usp.br>
Signed-off-by: Guilherme Gabriel <guilhermesangabriel@usp.br>
```

The maintainers responsible for the subsystem were **Deucher e Christian König** (and if any of you are reading this, thank you SO MUCH for your service!) in the *amd-gfx@lists.freedesktop.org* list. We sent the patch and waited for the results...

## The results

Yeah, it was **not accepted**. As Christian explained rather succinctly:

```
Well again this is not something we want to do.
Those functions are intentionally separated.

Regards,
Christian.
```

Oh well, fair enough. So the **duplication is actually a design decision!** We thought that this was a case of unaccounted code duplication, but it was not only accounted for buy may as well be a common practice in these drivers. We had no way to know it at the time, but it still was a good practice section and made us **want a little bit "more" for the next patch**. 