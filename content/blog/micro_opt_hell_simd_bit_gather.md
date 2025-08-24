+++
title = "SIMD Bit-Gather: X86 AVX2 Edition"
date = 2025-06-22
draft = false

[taxonomies]
categories = ["Random"]
tags = ["micro-optimization", "wasted-effort-stuff"]

[extra]
lang = "en"
toc = true
comment = false
copy = true
math = false
mermaid = false
outdate_alert = false
outdate_alert_days = 120
display_tags = true
truncate_summary = false
+++


Here is a small micro-optimization that I implemented for the new version of the terrain generator in Unity.
It is a simple bit-gather, but vectorized using ``X86 AVX2`` intrinsics.


For some reason, Burst compiler did not want to vectorize the ``Load4`` internal loop, so I wanted to see if using manually placed intrinsics would be faster.
It was faster by 3ms on the median time for a chunk of volume ``66^3``, but for a chunk with volume ``34^3``, this is most probably only a marginal improvement. I will most probably remove this for the sake of code cleanliness and upgradeability, since having raw intrinsics like that make modifying the code extremely more convoluted and annoying for such a small speedup. But, I will leave this here on my website just as a way to archive it because I still feel proud of such an optimization, even though it's not that good.

```cs, linenos, hl_lines=10 13 16 19 22-23
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private unsafe int CalculateMarchingCubesCode(uint3 position, int baseIndex) {
    // https://docs.unity3d.com/Packages/com.unity.burst@1.4/manual/docs/CSharpLanguageSupport_BurstIntrinsics.html
    // I LOVE MICROOPTIMIZATIONS!!! I LOVE DOING THIS ON A WHIM WITHOUT ACTUALLY TRUSTING PROFILER DATA!!!!
    // I actually profiled this and it is actually faster. Saved 3ms on the median time. Pretty good desu
    if (X86.Avx2.IsAvx2Supported) {
        int4 indices = (offsets[0] + new int4(baseIndex));
        int4 indices2 = (offsets[1] + new int4(baseIndex));

        v256 indices_v256 = new v256(indices.x, indices.y, indices.z, indices.w, indices2.x, indices2.y, indices2.z, indices2.w);

        // divide by 32
        v256 component_v256 = X86.Avx2.mm256_srli_epi32(indices_v256, 5);

        // modulo by 32
        v256 shift_v256 = X86.Avx2.mm256_and_si256(indices_v256, new v256(31u));

        // fetch the uints using indices
        v256 uints_v256 = X86.Avx2.mm256_i32gather_epi32(bits.GetUnsafeReadOnlyPtr(), component_v256, 4);

        // "shift" and "and"
        v256 shifted_right_v256 = X86.Avx2.mm256_srlv_epi32(uints_v256, shift_v256);
        v256 anded_v256 = X86.Avx2.mm256_and_si256(shifted_right_v256, new v256(1u));

        // check if the bits are set
        uint4 sets1 = new uint4(anded_v256.UInt0, anded_v256.UInt1, anded_v256.UInt2, anded_v256.UInt3);
        uint4 sets2 = new uint4(anded_v256.UInt4, anded_v256.UInt5, anded_v256.UInt6, anded_v256.UInt7);
        return math.bitmask(sets1 == 0) | (math.bitmask(sets2 == 0) << 4);
    } else {
        bool4 test = Load4(position, 0, baseIndex);
        bool4 test2 = Load4(position, 1, baseIndex);
        return math.bitmask(test) | (math.bitmask(test2) << 4);
    }
}


[MethodImpl(MethodImplOptions.AggressiveInlining)]
private bool4 Load4(uint3 position, int selector, int baseIndex) {
    uint4 indices = (uint4)(offsets[selector] + new int4(baseIndex));

    bool4 hits = false;

    for (int i = 0; i < 4; i++) {
        int index = (int)indices[i];

        int component = index / 32;
        int shift = index % 32;

        uint batch = bits[component];

        hits[i] = ((batch >> shift) & 1U) == 1;
    }

    return hits;
}
```