# `examples/simple/simple.cpp` Walkthrough

Source file: `/Volumes/Work/repos/clone/forks/llama.cpp/examples/simple/simple.cpp`

## 1) Args and defaults
- Sets defaults for `prompt`, `ngl`, and `n_predict`.
- Parses:
  - `-m` model path
  - `-n` number of generated tokens
  - `-ngl` number of layers to offload to GPU
- Remaining args are joined as prompt text.

## 2) Backend + model init
- `ggml_backend_load_all()` loads available backends.
- `llama_model_default_params()` creates model params.
- `model_params.n_gpu_layers = ngl` configures offload policy.
- `llama_model_load_from_file(...)` loads the GGUF model.

## 3) Tokenization
- Gets vocab: `llama_model_get_vocab(model)`.
- Uses two-pass tokenization:
  - first call with output buffer `NULL` to get required token count
  - second call writes token IDs into `prompt_tokens`

## 4) Context init
- `llama_context_default_params()` creates defaults.
- `ctx_params.n_ctx = n_prompt + n_predict - 1`
- `ctx_params.n_batch = n_prompt`
- `ctx_params.no_perf = false` enables perf reporting.
- `llama_init_from_model(model, ctx_params)` creates runtime context.

## 5) Sampler init
- Creates sampler chain with `llama_sampler_chain_init(...)`.
- Adds greedy sampler (`llama_sampler_init_greedy()`), so it always picks highest-prob token.

## 6) Prompt echo
- Iterates prompt token IDs, converts each to text piece via `llama_token_to_piece(...)`, and prints it.

## 7) First batch and encoder path
- Creates first batch from full prompt: `llama_batch_get_one(prompt_tokens.data(), prompt_tokens.size())`.
- If model has encoder (`llama_model_has_encoder(model)`):
  - runs `llama_encode(ctx, batch)`
  - switches to decoder start token batch

## 8) Main generation loop
- `llama_decode(ctx, batch)` evaluates current tokens.
- `llama_sampler_sample(smpl, ctx, -1)` samples next token from latest logits.
- Stops on end-of-generation token (`llama_vocab_is_eog(...)`).
- Prints sampled token and sets next batch to one token (`llama_batch_get_one(&new_token_id, 1)`).

## 9) Perf + cleanup
- Prints throughput and perf stats (`llama_perf_sampler_print`, `llama_perf_context_print`).
- Frees sampler, context, and model.

---

## Clarifications from follow-up questions

### `prompt.size()`
`std::string::size()` returns number of bytes (`char` count), not Unicode character count.

### Why `-1` in `n_ctx = n_prompt + n_predict - 1`
Last sampled token does not need a subsequent decode step, so only `n_predict - 1` generated tokens must fit into decode-side context growth.

### `for (auto id : prompt_tokens)`
This is a range-based for loop. `auto` is inferred as `llama_token`.

### What is `batch` in `llama_decode`
`llama_batch` is the per-call input package (tokens/embeddings + sequence metadata) consumed by `llama_decode`.
In this example:
- first call: full prompt batch
- later calls: single sampled token batch
