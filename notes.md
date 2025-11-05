To run the benchmark:
```
 python benchmarks/benchmark_serving_structured_output.py \
        --backend cerebras-chat \
        --model llama-3.3-70b \
        --dataset longcontext-qa \
        --structured-output-ratio 1.0 \
        --request-rate 5 \
        --num-prompts 30
```