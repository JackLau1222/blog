---
slug: ffmpeg-feedback-filter-bug-fix
title: FFmpeg中的feedback滤镜死循环bug修复
authors: [jacklau]
tags: [ffmpeg, bug-fix, filter, c]
---

先介绍一下feedback filter的作用

<!--truncate-->

参考官方文档 [https://ffmpeg.org/ffmpeg-filters.html#toc-feedback](https://ffmpeg.org/ffmpeg-filters.html#toc-feedback)

> Blur only top left rectangular part of video frame size 100x100 with gblur filter.

>`[in][blurin]feedback=x=0:y=0:w=100:h=100[out][blurout];[blurout]gblur=8[blurin]`

feedback可以通过x，y，w，h参数选中原画面某一部分来应用别的滤镜（例子中gblur是模糊效果），其他部分保持原状

###  feedback的实现也非常有趣，是一个嵌套循环，还是拿gblur举例子：

1. feedback 先拿到[in]即原输入，然后压入fifo中缓存 并根据x，y，w，h参数裁切部分画面喂给[blurout]

2. gblur滤镜中[blurout]有了帧，于是开始处理并把结果给到[blurin]
3. 又回到feedback，此时fifo缓存（存原完整画面）和[blurin]（部分经二次滤镜处理的画面）都有数据了，就合成一帧输出给[out]
4. feedback索要下一帧[in]输入，继续循环前三个步骤


本来一切运行顺利，但是FFmpeg 8.0中一个新增的commit改变了这一切
```Diff

From 4440e499bae0235a8f4f4308c45be8f70f29ff2d Mon Sep 17 00:00:00 2001
From: Marton Balint <cus@passwd.hu>
Date: Sun, 22 Jun 2025 15:39:29 +0200
Subject: [PATCH] avfilter/avfilter: always forward request frame in
 filter_activate_default
Even if all inputs are blocked an activate callback should request a frame on
some if its inputs if a frame is requested on any of its outputs.

Signed-off-by: Marton Balint <cus@passwd.hu>
---
 libavfilter/avfilter.c | 11 +++++++++++
 1 file changed, 11 insertions(+)
diff --git a/libavfilter/avfilter.c b/libavfilter/avfilter.c
index dd12533208..e03dc65fc6 100644
--- a/libavfilter/avfilter.c
+++ b/libavfilter/avfilter.c
@@ -1283,6 +1283,11 @@ static int filter_activate_default(AVFilterContext *filter)             return request_frame_to_filter(filter->outputs[i]);
         }
 }
+    for (i = 0; i < filter->nb_outputs; i++) {
+        FilterLinkInternal * const li = ff_link_internal(filter->outputs[i]);
+        if (li->frame_wanted_out)
+            return request_frame_to_filter(filter->outputs[i]);
+    }     return FFERROR_NOT_READY;
 }
 @@ -1416,6 +1421,12 @@ static int filter_activate_default(AVFilterContext *filter)
      Rationale: checking frame_blocked_in is necessary to avoid requesting
      repeatedly on a blocked input if another is not blocked (example:
 [buffersrc1][testsrc1][buffersrc2][testsrc2]concat=v=2).
+
+   - If an output has frame_wanted_out > 0 call request_frame().
+
+     Rationale: even if all inputs are blocked an activate callback should
+     request a frame on some if its inputs if a frame is requested on any of
+     its output.  */
  int ff_filter_activate(AVFilterContext *filter)
```

###  要看懂这个commit，需要先简单介绍一下filter框架的调度机制：

- FFmpeg中的多个filter调度是通过一种ready status机制实现的，谁的ready status等级更高，优先级更高，先处理谁
    
- 每个filter都有一个activate函数，默认是filter_activate_default，也可以拥有自定义的activate函数，这个activate函数和上面的ready status机制相辅相成，比如feedback函数想找gblur要数据，就可以request gblur，让gblur的ready status更高
    

再扩展下刚刚介绍的嵌套循环逻辑:

> 1. feedback 先拿到[in]即原输入，然后压入fifo中缓存 并根据x，y，w，h参数裁切部分画面喂给[blurout]

由于feedback缺少[blurin]不能产出完整的一帧，所以request [blurin]，filter框架会根据score机制跳入gblur滤镜处理流程

> 2. gblur滤镜中[blurout]有了帧，于是开始处理并把结果给到[blurin]

gblur使用默认的 filter_activate_default，gblur没有新的输入于是 request feedback喂下一帧
```c

    for (i = 0; i < filter->nb_outputs; i++) {
        FilterLinkInternal * const li = ff_link_internal(filter->outputs[i]);
        if (li->frame_wanted_out &&
            !li->frame_blocked_in) {
            return request_frame_to_filter(filter->outputs[i]);
        }
    }
```

ready status机制调度feedback activate函数，再次request [blurin] （以及[in]）

> 此处省略3，4步骤，聚焦于request逻辑
```c
    if (!s->feed || ctx->is_disabled) {
        if (ff_outlink_frame_wanted(ctx->outputs[0])) {
            ff_inlink_request_frame(ctx->inputs[0]);
            if (!ctx->is_disabled)
                ff_inlink_request_frame(ctx->inputs[1]);
            return 0;
        }
    }
```
ready status调度gblur的filter_activate_default，但由于request_frame_to_filter设置了li->frame_blocked_in ,所以不会重复request feedback，还记得上面feedback还request了[in]吗，所以ready status要调度原输入了，然后feedback就获得了新的一帧，一切可以继续进行。

###  FFmpeg 8.0引入的那个commit改变了filter_activate_default
```c
    for (i = 0; i < filter->nb_outputs; i++) {
        FilterLinkInternal * const li = ff_link_internal(filter->outputs[i]);
        if (li->frame_wanted_out &&
            !li->frame_blocked_in) {
            return request_frame_to_filter(filter->outputs[i]);
        }
    }
    for (i = 0; i < filter->nb_outputs; i++) {
        FilterLinkInternal * const li = ff_link_internal(filter->outputs[i]);
        if (li->frame_wanted_out)
            return request_frame_to_filter(filter->outputs[i]);
    }
```

即使li->frame_blocked_in被设置了（意味着gblur已经request feedback），新增的代码无视这一参数继续requst feedback

调度到feedback，requst gblur

调度到gblur，继续request feedback

...死循环了

###  解决方案

显然，新增的代码在公共部分，是有其应用场景的，那么新的filter框架就要求feedback这种特殊的filter在自己的activate函数中处理好request逻辑，避免死循环。

那么解决这个问题也很简单，feedback 原request代码之前增加对outputs[1] (blurout)检查，也就是检查feedback是否已经request gblur一次了，如果request过了也就意味着feedback需要更多帧输入，所以直接退出，让框架继续调度下一帧输入。

```c
    if (!s->feed || ctx->is_disabled) {
        if (!ctx->is_disabled && ff_outlink_frame_wanted(ctx->outputs[1])) {
            ff_inlink_request_frame(ctx->inputs[0]);
            return 0;
        }
        if (ff_outlink_frame_wanted(ctx->outputs[0])) {
            ff_inlink_request_frame(ctx->inputs[0]);
            if (!ctx->is_disabled)
                ff_inlink_request_frame(ctx->inputs[1]);
            return 0;
        }
    }
```

修复的patch已合并，master和8.0分支同步修复了。
https://code.ffmpeg.org/FFmpeg/FFmpeg/commit/3f0842294fbefcca32fdad6b644eae8c14f547e5
