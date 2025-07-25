### 🔀 [#505](https://github.com/ikawrakow/ik_llama.cpp/pull/505) - New IQ4_KT trellis implementation

| **Author** | `ikawrakow` |
| :--- | :--- |
| **State** | ❌ **Closed** |
| **Created** | 2025-06-08 |
| **Updated** | 2025-06-18 |

---

#### Description

This PR adds a new version of `IQ4_KT` based on a new trellis.

The new trellis generated `int8_t` values in `[-126...126]` instead of the original "3INST" version taken from the "3INTS" version taken from QTIP, which produces `fp16` values. The Gaussian distribution generated by the new trellis is much better that the original QTIP trellis. Sadly, this does not result in a lower quantization error. For `IQ4_KT`, the quantization error as measured by PPL is on par, or perhaps slightly lower than the exiting implementation on the main branch. But for `IQ2_KT` I consistently get a higher PPL, so for now this PR only changes the implementation to the new trellis for `IQ4_KT`. 

The main advantage of the new trellis is not a lower quantization error but a massively better performance, especially on the CPU. In addition, it allows for quantized GEMM and GEMV implementation on the GPU, which avoids numerical issues with DeepSeek models when dequantizing to `fp16`, along with a significantly better GEMM performance. 

Here some performance examples for LLaMA-3.1-8B
* Ryzen-7950X CPU: PP-512 = 273 t/s vs 133 t/s on main. TG-128 = 13.6 t/s vs 8.4 t/s on main
* M2-Max CPU: PP-512 = 121 t/s vs 75 t/s on main. TG-128 = 9.4 t/s vs 6.6 t/s on main
* RTX-4080 GPU: PP-512 = 8000 t/s vs 5800 t/s on main. TG-128 = 134 t/s vs 128 t/s on main.

What is the trick? If $v$ is an unsigned 32 bit integer and $A, B$ are unsigned 32-bit integer magic constants, in both cases we use $v \to A v + B$ to generate the next trellis value. The difference comes from the conversion of $v$ to an actual values to be used as a model weight:
* In the original QTIP trellis we have `s = (v & M_1) ^ M_2`, where $M_1$ and $M_2$ are suitable masks, and $s$ is another 32-bit unsigned integer. The used value is generated by viewing $s$ as two `fp16` values and using their sum
* In the new trellis we have `s = v & M`, $s$ is viewed as 4 `int8_t` values, and the result is their sum minus 126 for `M = 0x3f3f3f3f`, which can be computed very efficiently without requiring native `fp16` arithmetic support:
  - On CUDA one can use `__dp4a(s, 0x01010101, -126)`
  - On `Zen4` one can use `_mm256_dpbusd_epi32` to compute 8 values with a single instruction
  - Same on `NEON`, where one gets 4 values in a single instruction via `vdotq_s32`

---

#### 💬 Conversation

👤 **ikawrakow** commented the **2025-06-08** at **11:37:36**:<br>

Here a plot of the pdf generated via the the new trellis (black dots) and a Gaussian fit (red line)

![trellis](https://github.com/user-attachments/assets/ac35aae3-7308-4a86-a892-c68e35e60748)

One would get an even better Gaussian by summing the bytes of two trellis values (so, 8 `int8_t` values). But this only increases computation time without leading to a better quantization quality.

---

👤 **ubergarm** commented the **2025-06-08** at **19:45:04**:<br>

This looks interesting, was thinking to test out this `iq4_kt` against my [ubergarm/gemma-3-27B-it-qat-iq4_ks](https://github.com/ikawrakow/ik_llama.cpp/discussions/334#discussioncomment-13374007) which is supposedly pretty good according to the linked discussion comment.

I got it to compile CPU only e.g.

```bash
cmake -B build -DGGML_CUDA=OFF -DGGML_BLAS=OFF
cmake --build build --config Release -j $(nproc)
```

But not having luck getting it compile with CUDA e.g. variations of:
```bash
#cmake -B ./build -DGGML_CUDA=ON -DGGML_BLAS=OFF -DGGML_SCHED_MAX_COPIES=1 -DGGML_CUDA_F16=ON
#cmake -B ./build -DGGML_CUDA=ON -DGGML_BLAS=OFF -DGGML_SCHED_MAX_COPIES=1 -DGGML_CUDA_IQK_FORCE_BF16=1

rm -rf ./build/
cmake -B ./build -DGGML_CUDA=ON -DGGML_BLAS=OFF -DGGML_CCACHE=OFF
cmake --build ./build --config Release -j $(nproc)
```

There is a [warning about this switch/case fall through in `mmvq.cu`](https://github.com/ikawrakow/ik_llama.cpp/blob/ik/new_iq4kt/ggml/src/ggml-cuda/mmvq.cu#L527-L532) and a linker error about `mul_mat_q_case<(ggml_type)155> ...`

<details>

<summary>👈 Logs</summary>

```bash
# the warning
[ 45%] Building CXX object ggml/src/CMakeFiles/ggml.dir/iqk/iqk_quantize.cpp.o
[ 45%] Building C object ggml/src/CMakeFiles/ggml.dir/ggml-aarch64.c.o
/home/w/projects/ik_llama.cpp/ggml/src/ggml-cuda/mmvq.cu: In function ‘void ggml_cuda_op_mul_mat_vec_q_impl(ggml_backend_cuda_context&, ggml_type, int
64_t, int64_t, int64_t, int64_t, int64_t, int64_t, int64_t, const char*, const char*, float*, const char*, int64_t, int64_t, int64_t, int64_t, cudaStr
eam_t)’:
/home/w/projects/ik_llama.cpp/ggml/src/ggml-cuda/mmvq.cu:528:30: warning: this statement may fall through [-Wimplicit-fallthrough=]
  528 |             mul_mat_vec_iq4_kss_q8_1_cuda(src0_dd_i, src1_ddq_i, dst_dd_i, ids_data, ne00, row_diff, src1_padded_row_size, src1_ncols, nrows_d
st, ne2, nb02, nb12, nb2, ids_nb0, stream);
      |             ~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/home/w/projects/ik_llama.cpp/ggml/src/ggml-cuda/mmvq.cu:529:1: note: here
  529 |         case GGML_TYPE_IQ4_KT:
      | ^

# the error
[ 48%] Building CXX object src/CMakeFiles/llama.dir/llama-sampling.cpp.o
[ 48%] Linking CXX executable ../../bin/llama-gguf
/usr/bin/ld: ../../ggml/src/libggml.so: undefined reference to `void mul_mat_q_case<(ggml_type)155>(ggml_backend_cuda_context&, mmq_args const&, CUstr
eam_st*)'
collect2: error: ld returned 1 exit status
gmake[2]: *** [examples/gguf/CMakeFiles/llama-gguf.dir/build.make:98: bin/llama-gguf] Error 1
gmake[1]: *** [CMakeFiles/Makefile2:2643: examples/gguf/CMakeFiles/llama-gguf.dir/all] Error 2
gmake[1]: *** Waiting for unfinished jobs....
[ 48%] Linking CXX executable ../../bin/llama-gguf-hash
/usr/bin/ld: ../../ggml/src/libggml.so: undefined reference to `void mul_mat_q_case<(ggml_type)155>(ggml_backend_cuda_context&, mmq_args const&, CUstr
eam_st*)'
collect2: error: ld returned 1 exit status
gmake[2]: *** [examples/gguf-hash/CMakeFiles/llama-gguf-hash.dir/build.make:104: bin/llama-gguf-hash] Error 1
gmake[1]: *** [CMakeFiles/Makefile2:2510: examples/gguf-hash/CMakeFiles/llama-gguf-hash.dir/all] Error 2
[ 49%] Linking CXX shared library libllama.so
[ 49%] Built target llama
gmake: *** [Makefile:146: all] Error 2
```

</details>

For fun I tried compiling an earlier commit `fb776ab` closer to the CUDA implementation, but same error. I tried moving the duplicated `break;` which didn't effect the error. I tried rebasing it on top of main which has the `IQ2_M_R4` functionality but same error.

I see both `IQ4_KT = 155` and `GGML_TYPE_IQ4_KT 155` but don't know enough about c++ templates to figure out what I'm missing.

---

👤 **ikawrakow** commented the **2025-06-08** at **20:37:58**:<br>

The Ops are harmless, just forgotten to remove

On Sun, 8 Jun 2025 at 23:34, ubergarm ***@***.***> wrote:

> *ubergarm* left a comment (ikawrakow/ik_llama.cpp#505)
> <https://github.com/ikawrakow/ik_llama.cpp/pull/505#issuecomment-2954265218>
>
> Now that it seems to compile okay, giving it a try quantizing
> gemma-3-27B-it-qat-iq4_kt
>
> My first attempt threw an Oops Cluster N has no points but seems to keep
> going okay:
>
> [   4/ 808]                blk.0.ffn_gate.weight - [ 5376, 21504,     1,     1], type =   bf16, converting to iq4_kt .. cluster_points: Oops. Cluster 620 has no points:  0 3 2 1
> cluster_points: 1 out of 625 clusters dir not have any points
> cluster_points: Oops. Cluster 25 has no points:  1 2 1 0
> cluster_points: Oops. Cluster 124 has no points:  0 3 3 1
> cluster_points: Oops. Cluster 624 has no points:  0 0 3 1
> cluster_points: 3 out of 625 clusters dir not have any points
> size =   220.50 MiB ->    55.21 MiB
> [   5/ 808]                  blk.0.ffn_up.weight - [ 5376, 21504,     1,     1], type =   bf16, converting to iq4_kt .. size =   220.50 M
> iB ->    55.21 MiB
>
> Not sure what that means, so I'm making a new imatrix using the some extra
> stuff from exllamav3 on top of my usual to see if it still throws the Oops
> knowing it might be completely unrelated.
>
> Will update this with results...
>
> —
> Reply to this email directly, view it on GitHub
> <https://github.com/ikawrakow/ik_llama.cpp/pull/505#issuecomment-2954265218>,
> or unsubscribe
> <https://github.com/notifications/unsubscribe-auth/ALR6H4LJBPYLWEQ2RC437S33CSM6NAVCNFSM6AAAAAB6273QMKVHI2DSMVQWIX3LMV43OSLTON2WKQ3PNVWWK3TUHMZDSNJUGI3DKMRRHA>
> .
> You are receiving this because you authored the thread.Message ID:
> ***@***.***>
>