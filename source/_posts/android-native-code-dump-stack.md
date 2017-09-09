---
title: android native code dump stack
date: 2013-10-10 19:19:24
tags:
- android
---

<pre>
#include <corkscrew/backtrace.h>
void dumpStack(const char *const msg)
{
#define MAX_FRAMES 16
#define MAX_LINE 256
#define IGNORE_DEPTH 1
    backtrace_frame_t frames[MAX_FRAMES];
    backtrace_symbol_t symbols[MAX_FRAMES];
    int nframes, i;
    char bt_line[MAX_LINE];

    nframes = unwind_backtrace(frames, IGNORE_DEPTH, MAX_FRAMES);
    if (nframes > 0)
    {
        ALOGD("Stack trace: %s (%d frames)", msg, nframes);
        get_backtrace_symbols(frames, nframes, symbols);

        for (i = 0; i < nframes; i++)
        {
            format_backtrace_line(i, &frames[i], &symbols[i], bt_line, MAX_LINE);
            ALOGD(" %s", bt_line);
        }

        ALOGD("End of stack trace");
    }
    else
    {
        ALOGD("Stack trace unwinding failed: %s", msg);
    }
}
</pre>

link libcorkscrew in Android.mk
