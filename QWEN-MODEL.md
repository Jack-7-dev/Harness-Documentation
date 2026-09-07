# Running the Qwen3-Coder-30B-A3B model (Z440)

The model runs as a systemd service — it is **already running**, do not start it manually.

## Check it's up
```
systemctl status llama.service
curl -s localhost:8080/health
```

## Use it
OpenAI-compatible endpoint at `http://localhost:8080/v1`. Any OpenAI-client-shaped
tool works by pointing `base_url` there with any placeholder API key.

```
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen3-Coder-30B-A3B","messages":[{"role":"user","content":"hello"}]}'
```

## From another machine
The server has no auth of its own and CORS wide open — never expose port 8080
directly. Tunnel over SSH instead:
```
ssh -i ~/.ssh/z440_key -L 8080:localhost:8080 jack@192.168.1.120
```
Then point clients at `http://localhost:8080/v1` on the client side.

## Service control
```
sudo systemctl stop llama.service      # stop
sudo systemctl start llama.service     # start
sudo systemctl restart llama.service   # restart
journalctl -u llama.service -f         # tail logs
```
It's `enabled`, so it also comes back automatically on reboot — this box boots
headless (`multi-user.target`, no desktop), confirmed working end-to-end
2026-09-02.

## Expect on first request after every boot/restart
The first real inference request pays a one-time HIP JIT compile cost — prompt
processing runs at ~5-6 tok/s instead of the normal speed. This is expected,
not a bug. Generation speed after that is normal (~28 tok/s).

## VRAM
Model + 32K KV cache uses ~14.8-15.6 GB of the card's 17.1 GB. Non-model
driver overhead is ~600 MB headless (was ~800 MB with a desktop running).
Very little headroom (~900 MB-1.5 GB) for anything else GPU-side.

## Do not redo
- **Backend: ROCm**, not Vulkan — chosen after real workload-shaped
  benchmarking, not `llama-bench` defaults.
- **Runtime: llama.cpp built from source** (`~/SDK/llama.cpp/build-hip`) is
  the only runtime in use. Ollama was fully removed from this project.
