# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `nvidia_cuda, vulkan` ·
llama.cpp `b10488` · `threads=6` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 9.4 | 1.00x | 14% |
| 8 | 13.1 | 1.39x | 19% |
| 16 | 14.8 | 1.58x | 22% |
| 24 | 35.9 | 3.83x | 53% |
| 32 | 55.7 | 5.94x | 82% |
| 99 | 67.6 | 7.21x | 100% |

Best: `-ngl 99` at 67.6 tok/s
-- 7.21x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Full offload (`-ngl 99`, offload toàn bộ 35 layer của Gemma 4 E2B lên VRAM) đạt hiệu năng cao nhất trên máy này, đạt **67.6 tok/s** (tăng tốc **7.21x** so với thuần CPU ở 9.4 tok/s).

Ở các mức partial offload (`-ngl 8` đến `16`), mức tăng tốc khá khiêm tốn (1.39x đến 1.58x) vì ở mỗi bước sinh token, hệ thống vẫn phải truyền tải các tensor trạng thái trung gian qua bus PCIe giữa CPU và GPU. Khi `-ngl` đạt từ 24, 32 đến 99, hầu hết các layer và KV Cache đều nằm gọn trong VRAM của GPU RTX 3050, loại bỏ nghẽn băng thông PCIe và giúp các CUDA kernel thực thi liên tục ở tốc độ tối đa của VRAM.
