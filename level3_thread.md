# level3_thread.c

这个代码是多线程处理GEMM的关键代码,代码逻辑为将矩阵粗分割成为多个矩阵（尽可能保证其方阵提高效率），每个线程处理一个小矩阵，处理逻辑同level3.c中一致，分块，重排。

下面从函数调用路径来解释一下代码：

## 1.cmake函数
三个重要参数：nthreads(初始化为支持最大线程数)、nthreads_m(矩阵在行方向的线程数)、nthreads_n(矩阵在列方向的线程数)
```cpp
  /* Partitions in m should have at least switch_ratio rows */
  if (m < 2 * switch_ratio) {
    nthreads_m = 1;
  } else {
    nthreads_m = args -> nthreads;
    while (m < nthreads_m * switch_ratio) {
      nthreads_m = nthreads_m / 2;
    }
  }

  /* Partitions in n should have at most switch_ratio * nthreads_m columns */
  if (n < switch_ratio * nthreads_m) {
    nthreads_n = 1;
  } else {
    nthreads_n = (n + switch_ratio * nthreads_m - 1) / (switch_ratio * nthreads_m);
    if (nthreads_m * nthreads_n > args -> nthreads) {
      nthreads_n = blas_quickdivide(args -> nthreads, nthreads_m);
    }

    while (nthreads_m % 2 == 0 && n * nthreads_m + m * nthreads_n > n * (nthreads_m / 2) + m * (nthreads_n * 2)) {
      nthreads_m /= 2;
      nthreads_n *= 2;
    }
  }
```
这段代码主要是要实现nthreads_m,nthreads_n的参数大小。为了分出的块大小实现处理高效率，尽可能让分出的块大小接近方阵（表现在参数上就是 (n / nthreads_n) + (m / nthreads_m)最小化，方的东西周长最小），同时还需满足nthreads_m * nthreads_n<nthreads。

得到这俩个参数后将nthreads = nthreads_n * nthreads_m。随后调用gemm_driver函数：

## 2.gemm_driver函数:
重要参数是rang_M、range_N数组。函数根据nthreads_m,nthreads_n对矩阵进行切分，切分点存储在range_M、range_N中。（len(range_M)==nthreads_m）
```cpp
  num_parts = 0;
  while (m > 0){
    width = blas_quickdivide(m + nthreads_m - num_parts - 1, nthreads_m - num_parts);

    width = round_up(m, width, GEMM_PREFERED_SIZE);

    m -= width;

    if (m < 0) width = width + m;
    range_M[num_parts + 1] = range_M[num_parts] + width;

    num_parts ++;
  }
  for (i = num_parts; i < MAX_CPU_NUMBER; i++) {
    range_M[i + 1] = range_M[num_parts];
  }
```
这段代码主要是根据nthread_m,nthread_n这俩个参数，对矩阵进行一个thread级的划分，并且记录在range_m和range_n俩个数组中，最后调用inner_thread。

## 3.inner_thread函数(线程调度函数):
重要参数sa(a矩阵的buffer)、sb(b矩阵的buffer)、mypos(可以直接理解为thread_id)
```cpp
  mypos_n = blas_quickdivide(mypos, nthreads_m);  /* mypos_n = mypos / nthreads_m */
  mypos_m = mypos - mypos_n * nthreads_m;         /* mypos_m = mypos % nthreads_m */

  '''

    m_from = range_m[mypos_m + 0];
    m_to   = range_m[mypos_m + 1];

```
根据mypos(thread_id)就可以定位此线程所处理的矩阵位置mypos_m、mypos_n(列展开)。随后处理逻辑就与level3.c一致了，只不过处理的矩阵不是源矩阵而是经过cmake粗分割后的矩阵。

