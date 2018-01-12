---
title: using kmemleak
date: 2013-05-03 20:16:03
tags:
- kernel
---

## Overview
kmemleak use tricolor marking algorithm to track objects.

* White: objects that could be memory leaks    
* Grey: objects known not to be memory leaks    
* Black: objects that have no references to other objects in the white set

Kmemleak tracks objects allocated via

* kmalloc     
* kmem_cache_alloc  
* vmalloc
* alloc_bootmem     
* pcpu_alloc

Objects are no longer tracked once freed.

Remember it's not accurate, there're false negative/positive cases.

<!--more-->
## Usage

* Enable kernel config:

    * CONFIG_DEBUG_KMEMLEAK=y
    * CONFIG_DEBUG_KMEMLEAK_EARLY_LOG_SIZE=400
    * (optional)enable/disable at boot-time by passing "kmemleak=off" to the kernel command line.

* Start Analyzing from 
cat /sys/kernel/debug/kmemleak

* Advantage usage
change behavior runtime by echo something > /sys/kernel/debug/Kmemleak, the following parameters are supported:

        off           - disable kmemleak (irreversible)
        stack=on      - enable the task stacks scanning (default)
        stack=off     - disable the tasks stacks scanning
        scan=on       - start the automatic memory scanning thread (default)
        scan=off      - stop the automatic memory scanning thread
        scan=<secs>   - set the automatic memory scanning period in seconds
                      (default 600, 0 to stop the automatic scanning)
        scan          - trigger a memory scan
        clear         - clear list of current memory leak suspects, done by
                  marking all current reported unreferenced objects grey,
                  or free all kmemleak objects if kmemleak has been disabled.
        dump=<addr>     - dump information about the object found at <addr>

## An example:
<https://events.linuxfoundation.org/images/stories/pdf/lceu11_marinas.pdf>

## Reference:
<https://www.kernel.org/doc/Documentation/kmemleak.txt>

