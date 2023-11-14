---

layout: post

title: Optimizing Compilers are Cool (but not perfect)

---



### Note: This is a somewhat older post. The information in it may no longer be accruate.

### Update: As of `clang-1500.0.40.1` (2022), LLVM still does not seem to implement any of these optimizations.



I’ve long thought that programmers are too eager to apply optimizations that compilers can trivially generate, with the result that their code is needlessly opaque. For example, I’d be shocked if any optimizing compiler didn’t replace a divide by a power of 2 with a shift. As a result, I’ve never liked the practice of writing:

```c
int halfX = x >> 1;
int xMod128 = x & 127;
```

instead of

```c
int halfX = x / 2;
int xMod128 = x % 128;
```



But lately I’ve started to rethink that position. For reference, I used IAR Embedded Workbench for ARM version 7.40.3 targeting a Cortex-M4 (with a `VFPv4` single precision FPU), with the optimization set to high / speed (roughly equivalent to `-O2` in GCC). The compiler seems to handle simple methods pretty well. Here are two methods. They convert [fixed point values](https://en.wikipedia.org/wiki/Fixed-point_arithmetic) with, respectively, 15 digits and 14 digits after the radix point into single-precision floating point values.

```c
float32_t q0115ToFloat(int16_t q0115Value) {
 return q0115Value / 32768.f;
}
float32_t q0214ToFloat(int16_t q0214Value) {
 return q0214Value / 16384.f;
}
```

And here’s the generated assembler:

```assembly
00000000 :
0: ee00 0a10 vmov s0, r0
4: eeba 0a60 vcvt.f32.s16 s0, s0, #15
8: 4770 bx lr
0000000a :
a: ee00 0a10 vmov s0, r0
e: eeba 0a41 vcvt.f32.s16 s0, s0, #14
12: 4770 bx lr
```



This is a pretty useful optimization, The compiler realized that the division could be accomplished with a single [VCVT](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0553a/CHDJAEDB.html) instruction, instead of performing a `VCVT` followed by a multiply (or even a divide, if it was really not helping at all). And this conversion happens all the time. For example, here’s the source for the [ARM CMSIS](http://www.arm.com/products/processors/cortex-m/cortex-microcontroller-software-interface-standard.php) DSP library method, `arm_q15_to_float`, which is just an unrolled loop executing the above method (some edits to remove irrelevant code):

```c
void _arm_q15_to_float(
 q15_t * pSrc,
 float32_t * pDst,
 uint32_t blockSize)
{
 q15_t *pIn = pSrc; /* Src pointer */
 uint32_t blkCnt; /* loop counter */
 /* Run the below code for Cortex-M4 and Cortex-M3 */
 /*loop Unrolling */
 blkCnt = blockSize >> 2u;
 /* First part of the processing with loop unrolling. Compute 4 outputs at a time.
 ** a second loop below computes the remaining 1 to 3 samples. */
 while(blkCnt > 0u)
 {
 /* C = (float32_t) A / 32768 */
 /* convert from q15 to float and then store the results in the destination buffer */
 *pDst++ = ((float32_t) * pIn++ / 32768.0f);
 *pDst++ = ((float32_t) * pIn++ / 32768.0f);
 *pDst++ = ((float32_t) * pIn++ / 32768.0f);
 *pDst++ = ((float32_t) * pIn++ / 32768.0f);
 /* Decrement the loop counter */
 blkCnt--;
 }
 /* If the blockSize is not a multiple of 4, compute any remaining output samples here.
 ** No loop unrolling is used. */
 blkCnt = blockSize % 0x4u;
 while(blkCnt > 0u)
 {
 /* C = (float32_t) A / 32768 */
 /* convert from q15 to float and then store the results in the destination buffer */
 *pDst++ = ((float32_t) * pIn++ / 32768.0f);
 /* Decrement the loop counter */
 blkCnt--;
 }
}
```

How useful is the optimization? I threw together a quick benchmark, which involved filling a 2048-length array of `int16_t` with random values, and then converting it to `float32_t` using the above library method. I ran the tests on an [STM32F411CC](http://www.st.com/web/catalog/mmc/FM141/SC1169/SS1577/LN1877/PF260526) processor, running at 100Mhz. For the first, I naively executed a division for every sample (which turns into a `VCVT` followed by a `VDIV`). For the second, I replaced the division by a constant by a multiply by the inverse (which turns into a `VCVT` followed by a `VMUL`). For the third, I replaced the entire thing with a single `VCVT`.

| Generated Code                                                                                                               | Throughput (ticks / sample) |
| ---------------------------------------------------------------------------------------------------------------------------- | --------------------------- |
| VCVT followed by [VDIV](http://infocenter.arm.com/help/topic/com.arm.doc.dui0553a/CHDIDCBF.html)                             | 20.25                       |
| VCVT followed by [VMUL](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0553a/CHDHFGFD.html) by 1.f / 32768.f | 9.44                        |
| VCVT only                                                                                                                    | 7.37                        |

So division is really expensive, which isn’t all that surprising. But even a single extraneous multiply results in a 28% decrease in throughput.



So why the ambivalence in the title? Well, one day I decided that I’d like more than one bit of precision before the radix. No problem. I just copied the existing method and replaced the constant by `16384.f` (which is `2^14`):

```c
void _arm_q14_to_float(
 q15_t * pSrc,
 float32_t * pDst,
 uint32_t blockSize)
{
 q15_t *pIn = pSrc; /* Src pointer */
 uint32_t blkCnt; /* loop counter */
 /* Run the below code for Cortex-M4 and Cortex-M3 */
 /*loop Unrolling */
 blkCnt = blockSize >> 2u;
 /* First part of the processing with loop unrolling. Compute 4 outputs at a time.
 ** a second loop below computes the remaining 1 to 3 samples. */
 while(blkCnt > 0u)
 {
 /* C = (float32_t) A / 16384 */
 /* convert from q15 to float and then store the results in the destination buffer */
 *pDst++ = ((float32_t) * pIn++ / 16384.0f);
 *pDst++ = ((float32_t) * pIn++ / 16384.0f);
 *pDst++ = ((float32_t) * pIn++ / 16384.0f);
 *pDst++ = ((float32_t) * pIn++ / 16384.0f);
 /* Decrement the loop counter */
 blkCnt--;
 }
 /* If the blockSize is not a multiple of 4, compute any remaining output samples here.
 ** No loop unrolling is used. */
 blkCnt = blockSize % 0x4u;
 while(blkCnt > 0u)
 {
 /* C = (float32_t) A / 16384 */
 /* convert from q15 to float and then store the results in the destination buffer */
 *pDst++ = ((float32_t) * pIn++ / 16384.0f);
 /* Decrement the loop counter */
 blkCnt--;
 }
}
```



And as we determined earlier, a division by `16384.f` turns into a single `VCVT`. So I spun it up, and… **throughput decreased by about 30%!** . What happened? Here’s the disassembled output (Address `0x21c` corresponds to the start of the first loop body, and `s0` contains the constant `16384.f`):

<pre><code>
00000208 :
 208: b430 push {r4, r5}
 20a: 0893 lsrs r3, r2, #2
 20c: f000 80db beq.w 3c6
 210: f013 0403 ands.w r4, r3, #3
 214: ed9f 0aff vldr s0, [pc, #1020] ; 7f4
 218: f000 802b beq.w 272
 21c: f930 5b02 ldrsh.w r5, [r0], #2
 220: ee00 5a90 vmov s1, r5
<mark> 224: eef8 0ae0 vcvt.f32.s32 s1, s1 </mark>
<mark> 228: ee60 0a80 vmul.f32 s1, s1, s0 </mark>
 22c: edc1 0a00 vstr s1, [r1]
 230: f930 5b02 ldrsh.w r5, [r0], #2
 234: ee00 5a90 vmov s1, r5
<mark> 238: eef8 0ae0 vcvt.f32.s32 s1, s1 </mark>
<mark> 23c: ee60 0a80 vmul.f32 s1, s1, s0 </mark>
 240: edc1 0a01 vstr s1, [r1, #4]
 244: f930 5b02 ldrsh.w r5, [r0], #2
 248: ee00 5a90 vmov s1, r5
<mark> 24c: eef8 0ae0 vcvt.f32.s32 s1, s1 </mark>
<mark> 250: ee60 0a80 vmul.f32 s1, s1, s0 </mark>
 254: edc1 0a02 vstr s1, [r1, #8]
 258: f930 5b02 ldrsh.w r5, [r0], #2
 25c: ee00 5a90 vmov s1, r5
<mark> 260: eef8 0ae0 vcvt.f32.s32 s1, s1 </mark>
<mark> 264: ee60 0a80 vmul.f32 s1, s1, s0 </mark>
 268: edc1 0a03 vstr s1, [r1, #12]
 26c: 3110 adds r1, #16
 26e: 1e64 subs r4, r4, #1
 270: d1d4 bne.n 21c
 ;...
 ; lots more, but you get the idea
</code></pre>



What the hell? For some reason, the compiler decided to emit a `VCVT` followed by a `VMUL` for the divisions by `16384.f`, instead of a single `VCVT` (see the highlighted text). So here’s what the compiler does, as best as I can understand:

- Replace a single divide by `32768.f` with a single `VCVT`, **check**
- Replace a loop of divides by `32768.f` with a loop of `VCVT`s, **check**
- Replace a single divide by `16384.f` with a single `VCVT`, **check**
- Replace a loop of divides by `16384.f` with a loop of `VCVT`s, **no!**

I’m at a loss to explain why it did that, and I am certainly going to start paying closer attention to the instructions that the compiler emits, especially in hot loops.

## Postscript

For those wondering how other toolchains do, here’s the full table. A check indicates that the compiler emitted only a single `VCVT` per conversion.

| Toolchain | Single Q1.15 | Single Q2.14 | Batch Q1.15 | Batch Q2.14 |
| --------- | ------------ | ------------ | ----------- | ----------- |
| IAR       | ✔            | ✔            | ✔           | ✘           |
| GCC       | ✔            | ✔            | ✘           | ✘           |
| Clang     | ✘            | ✘            | ✘           | ✘           |

#### Versions

IAR: EWARM 7.40.3  
GCC: 7.3.1 20180622  
Clang: 900.0.39.2
