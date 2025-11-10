To run the long context qa benchmark:
```
 python benchmarks/benchmark_serving_structured_output.py \
        --backend cerebras-chat \
        --model <model-string> \
        --dataset longcontext-qa \
        --max-concurrency <#concurrent-requests> \
        --num-prompts <any number between 1 and 346 (dataset size)> \
        --no-structured-output \
        --output-len <output-length> \
        --temperature <temperature> \
        --save-results
```
To run the random dataset benchmark:
```
python benchmarks/benchmark_serving_structured_output.py \
        --backend cerebras-chat \
        --model llama3.1-8b \
        --dataset random \
        --max-concurrency 1 \
        --num-prompts 100 \
        --random-input-len 500 \
        --output-len 100 \
        --temperature 0.1 \
        --save-results
```
