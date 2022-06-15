---
title: "Why Is AVX 512 Useful for RPCS3?"
date: 2022-06-15T09:51:53-04:00
draft: false
---

It's often said that the importance of the difference between instruction sets on modern computers [is overblown,](https://chipsandcheese.com/2021/07/13/arm-or-x86-isa-doesnt-matter/) and it's hard to actually disagree with this observation. Since 90% of the average program is made up of basic ALU instructions, loads and stores, and branch instructions, and given the fact that the gap between different instruction sets is very small in terms of these basic instructions, it's easy to agree with this conclusion.

But the PS3 emulator [RPCS3](https://rpcs3.net/) isn't an average program. Even if you haven't used the emulator itself, you may have heard RPCS3 brought up as an example of consumer software that can take advantage of AVX-512. In this blog post I'll cover why exactly the new instructions and features introduced in the AVX-512 family are so useful for PS3 emulation. In [some situations](https://github.com/RPCS3/rpcs3/pull/12209) use of 512 bit instructions can be profitable for RPCS3, but in this post I'll be covering why the new instructions introduced are useful at 128 and 256 bit lengths.

# Larger register file

The AVX-512 instruction set comes with a register file 4 times the size of that in AVX2. The registers have doubled in width from 256 to 512 bits, and the number of registers available to the programmer have doubled from 16 to 32 as well.

The PS3s [SPUs](https://en.wikipedia.org/wiki/Cell_(microprocessor)#Synergistic_Processing_Elements_(SPE)) have a register file of 128 individual 128 bit wide registers. It doesn't include any general purpose registers like a typical CPU, and even memory addresses will live in these registers, despite the fact that an address will only use 32 of the 128 bits available. Since the SPUs do not implement [register renaming](https://en.wikipedia.org/wiki/Register_renaming) or [out of order execution](https://en.wikipedia.org/wiki/Out-of-order_execution) it's essential that loops on the SPUs are unrolled to actually take advantage of the two execution pipes available on the hardware.

Since an unrolled loop on the SPUs may use upwards of 60 registers, this poses a problem for emulators. The solution is to spill out registers to memory when too many are used to fit on the host CPUs registers, but this comes at a cost in terms of performance. The extra registers introduced in AVX-512 help mitigate this issue, though they don't solve it either.

# New forms of old instructions

AVX-512 doesn't just add new instructions, it also adds new encodings for old instructions as well. Many instructions which supported a memory operand now support the "embedded broadcast" feature, which allows a 32 or 64 bit value from memory to be broadcast to all elements of a vector before use. 

Another useful feature in the new encoding is the "disp8" compressed displacement. In normal x86-64 instructions an embedded displacement can be encoded in just one byte if the displacement is between -128 and 127. If the displacement exceeds this size a full 4 byte displacement is required in the encoding, even if the displacement is still relatively small, like 256. With disp8 compression the displacement can be encoded with a single byte if the displacement is a multiple of the vector size, and that multiple is between -128 and 127. For instance a displacement of 256 for a 128 bit/16 byte vector would be encoded as 16 * 16. 

Unfortunately the new EVEX encoding requires another byte over the old VEX encoding that was used by AVX instructions, so only 2 bytes are saved over a VEX encoded instruction when disp8 compression is used. Additionally only EVEX encoded instructions can adress registers 16-31, and not all old AVX instructions received the new encoding. These instructions can still be used in AVX-512 code, but some complexity is added since they can't address all the registers.

# Mask registers

AVX-512 also adds new mask registers which can be optionally used with EVEX encoded instructions. There are new comparison instructions which generate a mask in the mask registers as the result of a comparison between vectors. When a mask register is used as an operand all of the elements not selected by the mask will either be zeroed or leave the existing value in the destination register untouched. There are 8 mask registers, through k0 - k7, however only k1 - k7 can be used to mask things out, as k0 implicitly behaves as if all elements are selected.

That means that a sequence like this in AVX:

```
vpcmpeqd xmm1, xmm2, xmm3
vpaddd xmm2, xmm2, xmm4
vpand xmm2, xmm1, xmm2

```

Can be reduced to something like this in AVX512:
```
vpcmpeqd k1, xmm2, xmm3
vpaddd xmm2{k1}{z}, xmm2, xmm4
```

On current Intel CPUs instructions with a mask register as an argument have slightly longer latency, however it does not add any extra uops, which is more important than added latency on CPUs with out of order execution. Additionally the ability to save masks in the mask registers instead of in the vector registers should help free up more vector registers, effectively making the AVX-512 register file more than twice as big as the AVX2 one.



But enough about old instructions and the new ways to use them. How useful are the new instructions in AVX-512 for RPCS3?

# VPLZCNTD

This instruction counts the leading zeroes of each 32 bit element of the input vector. This just so happens to be exactly what the SPU instruction `clz` does.

# VPOPCNTB

This instruction counts the number of ones in each 8 bit element of the input vector. This just so happens to be exactly what the SPU instruction `cntb` does.

# VDBPSADBW

This instruction is a complicated one, intended for accelerating [sum of absolute differences](https://en.wikipedia.org/wiki/Sum_of_absolute_differences) calculations. This instruction performs many absolute difference calculations in parallel, before horizontally summing each block of 4 bytes together to create 16 bit results.

By using an input of all zeroes for the second input vector, we can abuse this instruction to emulate a horizontal addition instruction. Since the absolute difference between any given number and 0 will simply be our given number, this instruction becomes useful for performing horizontal addition on blocks of 4 bytes. This is useful for emulating the SPU instruction `sumb`.

# VPROLVD/VPROLD

These instructions rotate 32 bit elements in a vector by a specified value. This is a match for the `rot` and `roti` SPU instructions.

# VPTERNLOG

`vpternlog` is an instruction which can greatly simplify bitwise operations. The instruction takes [an immediate byte](https://www.officedaytime.com/simd512e/simdimg/ternlogcalc.html) which controls its operation. For instance the expression `(a & c) | (b & ~c)` can be realized in just one instruction by utilizing `vpternlog`. This particular expression will select bits from either vector based off the contents of `c`. This is useful for emulating the SPU instruction `selb`.

Not only is this instruction invaluable for emulating any bitwise instruction for which no previous analogue existed in x86, it's also invaluable for combining sequences of simpler bitwise instructions into smaller sequences. This instruction is very versatile, and is likely to become a favorite of any SIMD programmers who try writing AVX-512 software. The only drawback to this instruction is the dependency on the output register. It'd be convenient to have a 2 input version of this instruction as well when only 2 inputs are needed.

# VPDPWSSD

This instruction is a fusion of 2 existing x86 instructions, `vpmaddwd` and `vpadd`. `vpmaddwd` is a very peculiar instruction, which multiplies pairs of 16 bit numbers vertically, before summing the results horizontally. The `vpdpwssd` behaves the same, except it adds the result to the destination register as well.

Since the SPUs only support 16 bit multiplication, `vpmaddwd` is useful for emulating a number of the instructions in the `mpy` family of SPU instructions. Several of the `mpy` family instructions such as `mpya` add to an accumulator register as well, which makes `vpdpwssd` suitable for emulating these instructions.

This instruction was initially made available as part of AVX512-VNNI in the Sunny cove based processors. However, it now also has an AVX encoded version as well, included on the Golden cove and Gracemont based processors. 

# VRANGEPS

When browsing through the [public SPU isa documentation](https://arcb.csc.ncsu.edu/~mueller/cluster/ps3/SDK3.0/docs/arch/SPU_ISA_v1.2_27Jan2007_pub.pdf) there's a curious little paragraph about something the documentation calls "extended-range floating point format". 

![Extended Range Floating Point](/ExtendedRangeFP.png)

What in the world is extended-range floating point? And what do they mean by "game applications assumes a single-precision floating-point format that is distinct from the IEEE 754 format commonly implemented on general-purpose processors."?

It turns out that the SPUs inherit the [bizarre floating point format](https://wiki.pcsx2.net/PCSX2_Documentation/Nightmare_on_Floating-Point_Street) that the PS2 used. This floating point format does not support `NaN` (not a number) or `INF` (infinity), and instead interprets these values as very large floating point numbers. Once a `NaN` or `INF` value is generated, it's quite difficult to get rid of as most operations after will return `NaN` or `INF`. For instance `INF * 0 = INF`. `INF * -0.2 = -INF`. `NaN * 3 = NaN`.

The only way to accurately emulate this behavior is to reimplement it in our own software floating point implementation. Unfortunately, software floating point is incredibly slow, so RPCS3 does not (at this time) have software floating point emulation of extended-range floats (referred to by RPCS3 as "XFloat").

One [possible solution](https://wiki.pcsx2.net/PCSX2_Documentation/What%27s_clamping%3F_And_why_do_we_need_it%3F) is also used by the PS2 emulator [PCSX2](https://pcsx2.net/). By clamping `INF` and `NaN` to the maximum and minimum values representable on x86 CPUs we can solve most floating point related issues without too much overhead. With AVX we need two instructions to clamp our floats down to something useable. We can use `vpminsd` with a value of `0x7f7fffff` along with `vpminud` with a value of `0xff7fffff`. We have to use the integer version of these instructions since the floating point min/max instructions will return NaN if either operand is NaN.

By using `vrangeps` we can realize a 1 instruction solution to this problem. This instruction provides a superset of the functionality offered by the existing floating point min/max instructions. By using the immediate byte `0x2` with the instruction it will take the absolute value of our inputs, then take the minimum of our inputs, and then copy the sign bit from the first input over to the result, allowing us to do both negative and positive clamping at the same time. Crucially, this instruction will also only return `NaN` if both inputs are `NaN`.

RPCS3 will use the clamping strategy to handle floating point numbers when you select the "Relaxed XFloat" or the "Approximate XFloat" options. When using the accurate XFloat option RPCS3 will attempt to emulate the extended-range floats via double precision floats. This can handle some cases that clamping cannot, but it's much slower than clamping, and there are some games which are broken with "Accurate XFloat", but not with "Approximate XFloat".

Finally, it's worth noting that double precision float instructions on the SPUs are actually IEEE compliant. Unlike on the PS2 where all of the floating point hardware has a non compliant implementation each piece of the PS3 hardware has its own behavior. The FPU on the main PowerPC core of the PS3 is IEEE compliant. However the vector floating point unit on that same core only implements enough to be compliant with Java's standard for floating point. Throw in the PS3's gpu which has its own floating point hardware, and you're looking at 5 separate pieces of floating point hardware which all have different behavior. What a mess.

# VGF2PAFFINEQB

The `vgf2paffineqb` is an instruction intended for accelerating cryptographic calculations, however many SIMD programmers have realized that the instruction is useful for non cryptographic purposes as well. For RPCS3 the ability to [shuffle bits](http://0x80.pl/articles/avx512-galois-field-for-bit-shuffling.html) is useful.

This bit shuffling capability is useful for emulating part of the `shufb` SPU instruction. `shufb` is similar in behavior to the x86 instruction `pshufb`. Both instructions will rearrange bytes based off indices provided in another vector. Both instructions also have special case inputs which will insert a constant value into the destination value, rather than taking a value from one of the input vectors.

The x86 instruction `pshufb` only has a single special case input:

```
1xxxxxxx = 0x00
```

The spu instruction `shufb` has 3 special case inputs:
```
10xxxxxx = 0x00
110xxxxx = 0xFF
111xxxxx = 0x80
```

Additionally, the spu instruction `shufb` indexes into the vector by reverse order compared to `pshufb`, and `shufb` also takes 3 inputs, 2 data vectors and 1 vector containing indices. `pshufb` only takes 2 input vectors, 1 data vector and 1 vector with indices.

Actually explaining how `vgf2paffineqb` is used here is out of scope for this article, and will likely be the next subject I blog about. The AVX2 implementation of `shufb` has a reciprocal throughput of 4, while the `vgf2paffineqb` version has a reciprocal throughput of 2.3. This isn't a great result considering that the PS3 has a reciprocal throughput of 1 for this instruction, and the machine runs at 3.2ghz. However only a fraction of `shufb` instructions will need the full emulation, thanks to many optimizations applied during common uses of the instruction.

# Myriad of new shuffle instructions

The PS3's SPUs only have a single shuffle instruction, `shufb`. On x86 we have many different shuffle and permute instructions for shuffling different sizes of data. However, all that's needed to shuffle any size of data around that's larger than a byte is a shuffle bytes instruction.

When the indices to the `shufb` instruction are constant we can use one of these instructions designed for shuffling larger data sizes around when appropriate. For instance the new `vpermt2d/vpermi2d` instructions are useful when shuffling 32 bit data around. Unfortunately not all of the new instructions equally fast. The `vpermt2w/vpermi2w` and `vpermt2b/vpermi2b` instructions would be useful for shuffling 16 bit and 8 bit data around, but the performance in current cpus isn't ideal, since these instructions require 3 uops.

Since RPCS3 uses [LLVM](https://en.wikipedia.org/wiki/LLVM) to generate code, it should be able to choose these new instructions when appropriate. Since LLVM has a model of which instructions are fast and slow on each CPU, it should be able to make use of the best instruction for each situation. When/if new CPUs make these instructions fast, LLVM should be ready to use the previously slow instructions as well.

# What does all of this mean for performance?

[![Comparison Between ISA targets for GOW3](/Gow3ComparisonSmall.png)](/Gow3Comparison.png)

Screenshots captured on an i9 12900K @5.2GHz with [AVX-512 enabled](https://www.reddit.com/r/rpcs3/comments/tqt1ko/clearing_up_some_avx512_misinformation_and_how_to/)

From left to right: SSE2, SSE4.1, AVX2/FMA, and icelake tier AVX-512.

The performance when targeting SSE2 is absolutely terrible, likely due to the lack of the `pshufb` instruction from [SSSE3](https://en.wikipedia.org/wiki/SSSE3). `pshufb` is invaluable for emulating the `shufb` instruction, and it's also essential for byteswapping vectors, something that's necessary since the PS3 is a big [endian](https://en.wikipedia.org/wiki/Endianness) system, while x86 is little endian.

The SSE4.1 target achieves an average of 160 FPS, while the AVX2/FMA target achieves an average of 190 FPS. This is a 18% improvement over the SSE4.1 target. AVX2 doesn't include many new instructions over SSE4.1, but it does include a new 3 operand form for instructions, which eliminates many register to register `mov` instructions. Crucially, all CPUs that support AVX2 also support [FMA](https://en.wikipedia.org/wiki/Multiply%E2%80%93accumulate_operation#Fused_multiply%E2%80%93add) instructions. FMA instructions aren't just faster than a chain of multiply + add instructions, but can also produce different results due to not rounding to single precision between the multiply and the add. Accurately emulating this without FMA instructions adds some overhead, and so native FMA operations help out quite a bit.

The icelake tier AVX-512 target hits a ludicrous 235 FPS average, 23% faster than the AVX2/FMA target. The sheer number of new instructions added in AVX-512 is so large that quite a number of them end up being useful for RPCS3. Unlike AVX2 which was mostly a straightforward extension of existing SSE instructions to 256 bits, AVX-512 includes a huge number of new features which are very useful for SIMD programming, even at lower bit widths. However, since intel chose to market AVX-512 with the -512 moniker, people who aren't familiary with the instruction set usually fixate on the 512 bit vector aspect of the instruction set.

# Conclusion

One may wonder why it's worth targeting such new instruction sets when only the newest, and fastest processors support these instructions. After all, even when targeting SSE4.1, the 12900K @5.2GHz is able to produce a very playable 160 FPS. However, not all CPUs that support AVX-512 are 8 core monsters clocked at 5.2GHz. Most CPUs that support AVX-512 today are either server CPUs or laptop CPUs. Since Intel had problems ramping production on their 10nm process, intel chose to first release their new AVX-512 supporting line of processors on laptops, which can get away with smaller die sizes. The use of AVX-512 on these lower end laptop chips isn't going to push performance into the 200s, like it does on the giant 12900K, but it may help framerates reach a more playable level.

The recently announced [Zen 4](https://www.anandtech.com/show/17441/amd-zen-4-update-8-to-10-ipc-uplift-25-more-perfperwatt-vcache-chips-coming) was announced to support AVX-512 instructions as well. Since it's likely that the successor to devices such as the [steam deck](https://store.steampowered.com/steamdeck) will use a Zen 4 based CPU, it's possible the number of people wanting to play games on a low end device that supports AVX-512 will increase significantly. Even when the target framerate is already achievable without AVX-512, enabling AVX-512 optimizations could improve battery life, or provide more TDP to the gpu which could enable gameplay at higher resolutions. I've personally already observed this phenomenon today on my Tigerlake based laptop. When targeting AVX-512 the CPU cores use 1W less, and the GPU uses 1W more, enabling higher framerates in RPCS3.

Outside of RPCS3 AVX-512 isn't widely used by many emulators, however the Arm recompiler [dynarmic](https://github.com/merryhime/dynarmic) can take advantage of many AVX-512 instructions as well. Dynarmic is used by the 3DS emulator [Citra](https://citra-emu.org/), the Nintendo Switch emulator [Yuzu](https://yuzu-emu.org/) as well as the PS Vita emulator [Vita3K](https://vita3k.org/). I'm not aware of any benchmarks comparing AVX2 targets vs AVX-512 for any of these emulators, but I would assume that the gap is smaller than it is with RPCS3, since Arm cores support both vector instructions as well as scalar instructions. Since the average game will spend more time executing scalar instructions than vector instructions, the potential gain from vector optimizations isn't huge. For RPCS3 a large reason why these vector optimizations are so effective is because the SPUs only support operations on vector registers, and so any time spent emulating the SPUs is spent executing vector instructions.

One emulator that would likely benefit greatly from AVX-512 optimizations is PCSX2. Since the PS2's [VUs](https://en.wikipedia.org/wiki/Emotion_Engine#Vector_processing_units) inspired much of the behavior and design for the SPUs, many of the optimizations which apply to RPCS3 should also apply to PCSX2. In particular, `vrangeps` should be helpful for improving their clamping code.

Hopefully this should clear up some of the confusion around why AVX-512 is useful for RPCS3. I didn't highlight every AVX-512 instruction that RPCS3 can take advantage of, but I did highlight the ones that I thought were most interesting.