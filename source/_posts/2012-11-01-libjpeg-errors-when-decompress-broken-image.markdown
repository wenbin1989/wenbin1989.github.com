---
layout: post
title: "小议libjpeg解压损坏文件时的错误处理"
date: 2012-11-01 22:48
comments: true
categories: [C/C++, libjpeg]
keywords: C, C++, libjpeg, 损坏, 解压, 读取, 错误, jpeg_finish_decompress, jpeg_destroy_decompress
description: libjpeg解压损坏文件时，可能的一些错误与处理方法。通过错误处理、jpeg_finish_decompress()函数、调用jpeg_destroy_decompress()之前是否有必要调用jpeg_finish_decompress()三个问题展开讨论。
---

本文主要介绍libjpeg解压损坏文件时，可能的一些错误与处理方法。libjpeg的常规使用请参阅文档或其他文章。

##问题1：错误处理##

使用libjpeg解压文件时难免产生错误，原因可能是图片文件损坏、io错误、内存不足等等。
默认的错误处理函数会调用exit()函数，导致整个进程结束，这对用户来说是非常不友好的。我们需要注册自定义错误处理函数，改变此行为。

libjpeg采用c语言的setjmp/longjmp机制实现错误处理，首先需要包含以下头文件：
``` c
#include <setjmp.h>
```

定义错误处理结构体：
``` c
struct my_error_mgr {
  struct jpeg_error_mgr pub;	/* "public" fields */

  jmp_buf setjmp_buffer;	/* for return to caller */
};

typedef struct my_error_mgr * my_error_ptr;
```

实现错误处理函数：
``` c
METHODDEF(void)
my_error_exit (j_common_ptr cinfo)
{
  /* cinfo->err really points to a my_error_mgr struct, so coerce pointer */
  my_error_ptr myerr = (my_error_ptr) cinfo->err;

  /* Always display the message. */
  /* We could postpone this until after returning, if we chose. */
  (*cinfo->err->output_message) (cinfo);

  /* Return control to the setjmp point */
  longjmp(myerr->setjmp_buffer, 1);
}
```
以上代码只是简单的打印错误信息，之后调用longjmp，使控制流跳转到setjmp设置的代码块。
如果需要，还可通过cinfo->err->msg_code判断错误类型，进行进一步处理。

<!-- more -->

实现解压函数：
``` c
GLOBAL(int)
read_JPEG_file (char * filename)
{
  struct jpeg_decompress_struct cinfo;
  struct my_error_mgr jerr;
  /* More stuff */
  FILE * infile;		/* source file */

  /* In this example we want to open the input file before doing anything else,
   * so that the setjmp() error recovery below can assume the file is open.
   */
  if ((infile = fopen(filename, "rb")) == NULL) {
    fprintf(stderr, "can't open %s\n", filename);
    return 0;
  }

  /* We set up the normal JPEG error routines, then override error_exit. */
  cinfo.err = jpeg_std_error(&jerr.pub);
  jerr.pub.error_exit = my_error_exit;
  /* Establish the setjmp return context for my_error_exit to use. */
  if (setjmp(jerr.setjmp_buffer)) {
    /* If we get here, the JPEG code has signaled an error.
     * We need to clean up the JPEG object, close the input file, and return.
     */
    jpeg_destroy_decompress(&cinfo);
    fclose(infile);
    return 0;
  }

  /* ...
   * 读取数据
   * ...
   */

  /* This is an important step since it will release a good deal of memory. */
  jpeg_destroy_decompress(&cinfo);

  /* After finish_decompress, we can close the input file.
   * Here we postpone it until after no more JPEG errors are possible,
   * so as to simplify the setjmp error logic above.
   */
  fclose(infile);

  return 1;
}
```

以上代码重点在于，注册my_error_exit回调函数，之后调用setjmp函数设置返回代码块。
发生错误时，回调my_error_exit函数，在my_error_exit函数末尾，控制流跳转到setjmp设置的代码块，在此代码块完成释放资源等操作。
这样，即使发生错误，也不至于进程退出，造成用户困扰。


##问题2：关于jpeg_finish_decompress()函数##

网上几乎所有使用libjpeg解压图片的代码，在解压完成后都会调用以下代码：
``` c
/* Finish decompression */
jpeg_finish_decompress(&cinfo);
/* Release JPEG decompression object */
jpeg_destroy_decompress(&cinfo);

```

其中，jpeg_finish_decompress()函数的作用为：

* 检查jpeg对象状态是否正常，并将其设为解压完成状态
* 读取输入数据源中剩余数据，直至EOI标记（0xFF,0xD9，标记jpeg图片结束）
* 释放jpeg对象分配的工作内存

实现如下：
``` c
GLOBAL(boolean)
jpeg_finish_decompress (j_decompress_ptr cinfo)
{
  if ((cinfo->global_state == DSTATE_SCANNING ||
       cinfo->global_state == DSTATE_RAW_OK) && ! cinfo->buffered_image) {
    /* Terminate final pass of non-buffered mode */
    if (cinfo->output_scanline < cinfo->output_height)
      ERREXIT(cinfo, JERR_TOO_LITTLE_DATA);
    (*cinfo->master->finish_output_pass) (cinfo);
    cinfo->global_state = DSTATE_STOPPING;
  } else if (cinfo->global_state == DSTATE_BUFIMAGE) {
    /* Finishing after a buffered-image operation */
    cinfo->global_state = DSTATE_STOPPING;
  } else if (cinfo->global_state != DSTATE_STOPPING) {
    /* STOPPING = repeat call after a suspension, anything else is error */
    ERREXIT1(cinfo, JERR_BAD_STATE, cinfo->global_state);
  }
  /* Read until EOI */
  while (! cinfo->inputctl->eoi_reached) {
    if ((*cinfo->inputctl->consume_input) (cinfo) == JPEG_SUSPENDED)
      return FALSE;		/* Suspend, come back later */
  }
  /* Do final cleanup */
  (*cinfo->src->term_source) (cinfo);
  /* We can use jpeg_abort to release memory and reset global_state */
  jpeg_abort((j_common_ptr) cinfo);
  return TRUE;
}
```

从以上代码可以看到，jpeg_finish_decompress()函数是利用jpeg_abort()函数进行释放工作内存操作的。

解压损坏图片时，即使在调用jpeg_finish_decompress()之前的解压过程完全正常，调用jpeg_finish_decompress()时仍可能产生错误。
错误原因可能数据源结构损坏，导致丢失EOI标记，或有多个SOI标记等。
例如，解压下图时，调用jpeg_finish_decompress()之前完全正常，但调用jpeg_finish_decompress()时会产生错误：
*Invalid JPEG file structure: two SOI markers*

{% img https://lh5.googleusercontent.com/-2fC4tAPLYzU/UJKDhHuaEOI/AAAAAAAAAHE/kHoVUrhC2-w/s800/3.jpg %}

直觉上，如果在调用jpeg_finish_decompress()之前未发生错误，则直接利用所读取的图片数据用于显示等操作即可，
不必理会jpeg_finish_decompress()中可能产生的错误，进一步，甚至可能不必调用jpeg_finish_decompress()函数。
由此引出下一个问题：

##问题3：调用jpeg_destroy_decompress()之前是否有必要调用jpeg_finish_decompress()？##

由上文所述，调用jpeg_finish_decompress()的目的为：

1. 检查jpeg对象状态是否正常，并将其设为解压完成状态
2. 读取输入数据源中剩余数据，直至EOI标记（0xFF,0xD9，标记jpeg图片结束）
3. 释放jpeg对象分配的工作内存

对于目的1，如果不需检测jpeg对象状态，或者不再继续使用该对象，可直接忽略。

对于目的2，如果不需读取并消耗剩余输入数据源数据，可直接忽略。

对于目的3，即使不调用jpeg_finish_decompress()，jpeg对象分配的工作内存也会在调用jpeg_destroy_decompress()时被释放，不会造成内存泄露。
此外，如果还要继续使用jpeg对象，也可调用jpeg_abort()函数进行释放工作内存操作。

可简单做出如下结论：

* 如果只需得到解压数据，不再使用jpeg对象，则完全可以不必调用jpeg_finish_decompress()函数，直接调用jpeg_destroy_decompress()函数释放内存即可。
* 如果还需使用jpeg对象，但不在意对象状态，且忽略输入数据源剩余数据，可用jpeg_abort()代替jpeg_finish_decompress()，达到释放工作内存的目的。
* 如需检查jpeg对象状态，或者需要读取并消耗剩余输入数据源数据，则需调用jpeg_finish_decompress()函数。
此时需要注意jpeg_finish_decompress()可能产生的错误。
