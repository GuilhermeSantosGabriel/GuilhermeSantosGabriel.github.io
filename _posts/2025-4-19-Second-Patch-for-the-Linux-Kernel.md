---
layout: post
title:  "Second patch for the Linux Kernel: using guards instead of manual lock + unlocks"
---

**This is the second** of the two patches made in tandem with Gabriel Dimant and André Jun Hirata for the MAC0470 course on Free Software Development. This time we will be attempting to breathe new life to a file by updating the manual lock and unlock operations with the (kinda) new, shiny **guard!**

## The Subsystem

The subsystem that we chose was taken from a list made by the MAC0470 team of possible points of improvement. The file that we ended up tackling was part of the **amdgpu** driver, the driver from the drm subsystem that supports mostly (if not all) AMD Radeon GPUs (thank you amdgpu, my notebook depends on you!). This is the same subsystem that we touched upon our last kernel adventure.


## What was identified?

To understand the problem we first need to understand why someone would use locks. So...

### Why use locks?

The Operating System is known for working with concurrency. That means that we need to ensure that specific sections of code run **atomically**, as in indivisible. This prevents data inconsistency, race conditions and the dreadful deadlock (great game btw). The primitive *mutex* is a common way of assigning such areas:

```
struct mutex lock;   // declaring a mutex
mutex_init(&lock);   // dynamically initialising the mutex
mutex_lock(&lock);   // acquiring the mutex
...                  // critical section
mutex_unlock(&lock); // unlocking the mutex
```
(Example by https://flusp.ime.usp.br/courses/linux-kernel-patch-suggestions/#suggestion-11-replace-mutex_locklock-and-mutex_unlocklock-calls-with-guardmutexlock)

This works, but it depends on the user to **remember to unlock** after usage, sometimes in multiple places at once, leading to a specific design pattern: using it alongside **goto**. 

```
mutex_lock(&lock);

// DO A BUNCH OF STUFF AND MAYBE CALL goto unlock_return; IF RETURN NEEDED

unlock_return:
    mutex_unlock(&lock);
    return something;
```
goto has problems on its own, but this usage (and many more) may now be substituted with a new and simpler way using **guard**. Calling something like *guard(mutex)(&lock);* is essentially the same as using a mutex and unlocking it **at the end of the scope**, but simpler. This is already much better as it prevents user errors when forgetting to call *mutex_unlock(&lock)*, simplifying the aforementioned "goto" pattern and a bunch more usages. guards also come in a scoped form (if you need to unlock it before doing more stuff in the scope).

### goto HELL; & more

After running an obscure grep command that made all of us go "what but how" to list all the combined usages of "goto" and "mutex" we found literally thousand of files that could use a little bit of help. We settled for a file in the **power management module** of the amdgpu driver (found in *drivers/gpu/drm/amd/pm/*) named *amdgpu_dpm.c* that had a whopping 103 occurrences!

## What we ended up doing

In this file we changed three kinds of mutex_lock + mutex_unlock usages:

### simple lock and unlock right before return
These cases (most of the actual occurrences) consisted on simple functions that needed a protected region after a certain point:

```
@@ -46,10 +47,9 @@ int amdgpu_dpm_get_sclk(struct amdgpu_device *adev, bool low)
       if (!pp_funcs->get_sclk)
               return 0;

       mutex_lock(&adev->pm.mutex);
       ret = pp_funcs->get_sclk((adev)->powerplay.pp_handle,
                                low);
       mutex_unlock(&adev->pm.mutex);

       return ret;
```

 As the function ended right after the unlock, **these cases needed only a simple guard inplace of the mutex_lock, removing the mutex_unlock altogether.**

### simple lock and unlock followed by more code
This scenario looks much like the last one, but with the addition that after the unlock and before the return we actually have code that doesnt need to be protected:

```
@@ -126,9 +121,9 @@ int amdgpu_dpm_set_gfx_power_up_by_imu(struct amdgpu_device *adev)
       struct smu_context *smu = adev->powerplay.pp_handle;
       int ret = -EOPNOTSUPP;

       mutex_lock(&adev->pm.mutex);
       ret = smu_set_gfx_power_up_by_imu(smu);
       mutex_unlock(&adev->pm.mutex);

       msleep(10);
```

In these cases we simply use a **scoped_guard** and call it a day.

### goto (unlock + return) pattern
These functions use the pattern explained before:

```
@@ -80,13 +79,12 @@ int amdgpu_dpm_set_powergating_by_smu(struct amdgpu_device *adev,
       enum ip_power_state pwr_state = gate ? POWER_STATE_OFF : POWER_STATE_ON;
       bool is_vcn = block_type == AMD_IP_BLOCK_TYPE_VCN;

       mutex_lock(&adev->pm.mutex);

       if (atomic_read(&adev->pm.pwr_state[block_type]) == pwr_state &&
                       (!is_vcn || adev->vcn.num_vcn_inst == 1)) {
               dev_dbg(adev->dev, "IP block%d already in the target %s state!",
                               block_type, gate ? "gate" : "ungate");
               goto out_unlock;
       }

       switch (block_type) {
@@ -115,9 +113,6 @@ int amdgpu_dpm_set_powergating_by_smu(struct amdgpu_device *adev,
       if (!ret)
               atomic_set(&adev->pm.pwr_state[block_type], pwr_state);

out_unlock:
       mutex_unlock(&adev->pm.mutex);

       return ret;
}
```

Those cases were changed by simple having a guard instead of the mutex_lock and removing the goto altogether, substituting the goto calls for a simple return (as guard will take care of the unlocking action when leaving the scope) .

## Sending the patch

After these changes we started working on the patch. I was listed as co-developed-by, as my buddy Jun sent the patch itself. 

### THE PATCH MESSAGE:
```
drm/amd/pm: Use guard(mutex) instead of manual lock+unlock

Use guard() and scoped_guard() for handling mutex lock instead of
manually locking and unlocking the mutex. This prevents forgotten
locks due to early exits and removes the need of gotos.

Signed-off-by: Andre Jun Hirata <andrejhirata@usp.br>
Co-developed-by: Gabriel Dimant <gabriel.dimant@usp.br>
Signed-off-by: Gabriel Dimant <gabriel.dimant@usp.br>
Co-developed-by: Guilherme Gabriel <guilhermesangabriel@usp.br>
Signed-off-by: Guilherme Gabriel <guilhermesangabriel@usp.br>
```

The maintainer responsible for the subsystem was **Kenneth Feng** (and if any of you are reading this, thank you SO MUCH for your service!) in the *amd-gfx@lists.freedesktop.org* list. We sent the patch and waited for the results...

## The results

We... **dont have the results yet**. We hope this one ends up actually being accepted, and if so, we will keep on doing it to other files in the *drivers/gpu/drm/amd/pm/* directory. History will tell.