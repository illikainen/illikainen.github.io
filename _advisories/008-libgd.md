---
date: 2016-04-22
---
# CVE-2016-3074: libgd: signedness vulnerability

## Overview

libgd [1] is an open-source image library.  It is perhaps primarily used
by the PHP project.  It has been bundled with the default installation
of PHP since version 4.3 [2].

A signedness vulnerability (CVE-2016-3074) exist in libgd 2.1.1 which
may result in a heap overflow when processing compressed gd2 data.


## Details

4 bytes representing the chunk index size is stored in a signed integer,
chunkIdx[i].size, by `gdGetInt()` during the parsing of GD2 headers:

libgd-2.1.1/src/gd_gd2.c:
```c
 53 typedef struct {
 54     int offset;
 55     int size;
 56 }
 57 t_chunk_info;
```

libgd-2.1.1/src/gd_gd2.c:
```c
 65 static int
 66 _gd2GetHeader (gdIOCtxPtr in, int *sx, int *sy,
 67                int *cs, int *vers, int *fmt, int *ncx, int *ncy,
 68                t_chunk_info ** chunkIdx)
 69 {
...
 73     t_chunk_info *cidx;
...
155     if (gd2_compressed (*fmt)) {
...
163         for (i = 0; i < nc; i++) {
...
167             if (gdGetInt (&cidx[i].size, in) != 1) {
168                 goto fail2;
169             };
170         };
171         *chunkIdx = cidx;
172     };
...
181 }
```

`gdImageCreateFromGd2Ctx()` and `gdImageCreateFromGd2PartCtx()` then
allocates memory for the compressed data based on the value of the
largest chunk size:

libgd-2.1.1/src/gd_gd2.c:
```c
371|637     if (gd2_compressed (fmt)) {
372|638         /* Find the maximum compressed chunk size. */
373|639         compMax = 0;
374|640         for (i = 0; (i < nc); i++) {
375|641             if (chunkIdx[i].size > compMax) {
376|642                 compMax = chunkIdx[i].size;
377|643             };
378|644         };
379|645         compMax++;
...|...
387|656         compBuf = gdCalloc (compMax, 1);
...|...
393|661     };
```

A size of <= 0 results in `compMax` retaining its initial value during
the loop, followed by it being incremented to 1.  Since `compMax` is
used as the nmemb for `gdCalloc()`, this leads to a 1*1 byte allocation
for `compBuf`.

This is followed by compressed data being read to `compBuf` based on the
current (potentially negative) chunk size:

libgd-2.1.1/src/gd_gd2.c:
```c
339 BGD_DECLARE(gdImagePtr) gdImageCreateFromGd2Ctx (gdIOCtxPtr in)
340 {
...
413         if (gd2_compressed (fmt)) {
414
415             chunkLen = chunkMax;
416
417             if (!_gd2ReadChunk (chunkIdx[chunkNum].offset,
418                                 compBuf,
419                                 chunkIdx[chunkNum].size,
420                                 (char *) chunkBuf, &chunkLen, in)) {
421                 GD2_DBG (printf ("Error reading comproessed chunk\n"));
422                 goto fail;
423             };
424
425             chunkPos = 0;
426         };
...
501 }
```


libgd-2.1.1/src/gd_gd2.c:
```c
585 BGD_DECLARE(gdImagePtr) gdImageCreateFromGd2PartCtx (gdIOCtx * in, int srcx, int srcy, int w, int h)
586 {
...
713         if (!gd2_compressed (fmt)) {
...
731         } else {
732             chunkNum = cx + cy * ncx;
733
734             chunkLen = chunkMax;
735             if (!_gd2ReadChunk (chunkIdx[chunkNum].offset,
736                                 compBuf,
737                                 chunkIdx[chunkNum].size,
738                                 (char *) chunkBuf, &chunkLen, in)) {
739                 printf ("Error reading comproessed chunk\n");
740                 goto fail2;
741             };
...
746         };
...
815 }
```

The size is subsequently interpreted as a size_t by `fread()` or
`memcpy()`, depending on how the image is read:

libgd-2.1.1/src/gd_gd2.c:
```c
221 static int
222 _gd2ReadChunk (int offset, char *compBuf, int compSize, char *chunkBuf,
223            uLongf * chunkLen, gdIOCtx * in)
224 {
...
236     if (gdGetBuf (compBuf, compSize, in) != compSize) {
237         return FALSE;
238     };
...
251 }
```

libgd-2.1.1/src/gd_io.c:
```c
211 int gdGetBuf(void *buf, int size, gdIOCtx *ctx)
212 {
213     return (ctx->getBuf)(ctx, buf, size);
214 }
```


For file contexts:

libgd-2.1.1/src/gd_io_file.c:
```c
 52 BGD_DECLARE(gdIOCtx *) gdNewFileCtx(FILE *f)
 53 {
...
 67     ctx->ctx.getBuf = fileGetbuf;
...
 76 }
...
 92 static int fileGetbuf(gdIOCtx *ctx, void *buf, int size)
 93 {
 94     fileIOCtx *fctx;
 95     fctx = (fileIOCtx *)ctx;
 96
 97     return (fread(buf, 1, size, fctx->f));
 98 }
```


And for dynamic contexts:

libgd-2.1.1/src/gd_io_dp.c:
```c
 74 BGD_DECLARE(gdIOCtx *) gdNewDynamicCtxEx(int initialSize, void *data, int freeOKFlag)
 75 {
...
 95     ctx->ctx.getBuf = dynamicGetbuf;
...
104 }
...
256 static int dynamicGetbuf(gdIOCtxPtr ctx, void *buf, int len)
257 {
...
280     memcpy(buf, (void *) ((char *)dp->data + dp->pos), rlen);
...
284 }
```


## PoC

Against Ubuntu 15.10 amd64 running nginx with php5-fpm and php5-gd [3]:

```sh
$ python exploit.py --bind-port 5555 http://1.2.3.4/upload.php
[*] this may take a while
[*] offset 912 of 10000...
[+] connected to 1.2.3.4:5555
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

uname -a
Linux wily64 4.2.0-35-generic #40-Ubuntu SMP Tue Mar 15 22:15:45 UTC
2016 x86_64 x86_64 x86_64 GNU/Linux

dpkg -l|grep -E "php5-(fpm|gd)"
ii  php5-fpm       5.6.11+dfsg-1ubuntu3.1 ...
ii  php5-gd        5.6.11+dfsg-1ubuntu3.1 ...

cat upload.php
<?php
    imagecreatefromgd2($_FILES["file"]["tmp_name"]);
?>
```


## Solution

This bug has been fixed in git HEAD [4].


## References

1. <http://libgd.org/>
2. <https://en.wikipedia.org/wiki/Libgd>
3. <https://github.com/dyntopia/exploits/tree/master/CVE-2016-3074>
4. <https://github.com/libgd/libgd/commit/2bb97f407c1145c850416a3bfbcc8cf124e68a19>