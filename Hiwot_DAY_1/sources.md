## Canonical Sources Cited

1. DistServe: Prefill-Decode Introduction  
   https://adityashrishpuranik.com/writing/dist-serve-1-prefill-decode-introduction

2. LLM Inference Benchmarking - Measure What Matters (DigitalOcean)  
   https://www.digitalocean.com/blog/llm-inference-benchmarking

## Tool/Pattern Used by the Writer

- First-token timing pattern with a custom Hugging Face `TextStreamer` callback to estimate prefill (TTFT) vs decode (TPOT).
- Optional kernel-level validation with PyTorch Profiler (`torch.profiler`) and CUDA trace segmentation.
