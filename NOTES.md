# Notes for branch `fix-qwen3-vl-nested-video-markers` (commit 95eba1ac)

```text
CONTEXT NOTES (not part of the commit message; not a PR yet)

Issue
  vllm-project/vllm#69473: with transformers >= 5.10, the Qwen3-VL video
  prompt nests the per-frame <|vision_start|>/<|vision_end|> groups inside
  the chat template's outer pair, adding 2 tokens per video. Reported impact:
  Cosmos3-Super greedy output diverges from the official reference.

Root cause
  transformers 5.2.0 and 5.9.0 replace the whole
  <|vision_start|><|video_pad|><|vision_end|> triple. 5.10.0 is the first
  release where Qwen3VLProcessor overrides replace_video_token to expand only
  the bare <|video_pad|>, keeping the outer pair (checked in the 5.2.0, 5.9.0,
  5.10.0 and 5.20.2 wheels). vLLM's _expands_only_video_token() detected that
  override and switched its video target to the bare token, copying the
  nesting. vLLM applies its own prompt updates for video (the HF text
  expansion is not used), so this was vLLM's own choice.

Fix in this commit
  - Remove _expands_only_video_token() and the Cosmos3 Edge override.
  - In Qwen3VLMultiModalProcessor._apply_prompt_updates, when the request has
    videos, collapse each <|vision_start|><|video_pad|><|vision_end|> into a
    bare <|video_pad|>, then match [video_token_id].
  - Same idea as Lfm3OmniMultiModalProcessor._postprocess_prompt.
  - Wrapped and bare placeholders both produce the transformers 4.57.1 layout.

DUPLICATE-WORK WARNING (read before opening a PR)
  vllm-project/vllm#69474 (author four1er, opened 2026-11-23, open) fixes
  the same issue. vLLM's AGENTS.md says not to open a second PR for the same
  fix; if the approach is materially different, explain it in the issue first.
  Difference: #69474 always targets the full triple, so a bare <|video_pad|>
  prompt fails with
    PromptUpdateMismatchError: Failed to apply prompt replacement for
    mm_items['video'][0]
  (reproduced on CPU by patching that target in). main accepts bare
  placeholders today, and the GPU test
  tests/models/multimodal/generation/test_qwen3_vl.py::
  test_qwen3_vl_pruned_video_prefix_cache_hit builds its prompt that way.
  This commit keeps bare placeholders working.

Validation (CPU, transformers 5.20.2, torch 2.15.0, base f2c5fc20, fix 95eba1ac)
  - New test test_processor_video_prompt_does_not_nest_vision_markers:
    on main the wrapped case fails with the exact nested output from the
    issue; the bare case passes. With the fix both pass.
  - pytest tests/models/multimodal/processing/test_qwen3_vl.py
      74 passed, 2 warnings in 26.17s
  - test_common.py::test_processing_correctness[1.0-32-0.3-<model>]
    (cached vs uncached), 1 passed each:
      Qwen/Qwen3-VL-4B-Instruct 14.35s, Qwen/Qwen3.5-0.8B 10.78s,
      nvidia/Cosmos3-Edge 11.87s, nvidia/Cosmos3-Nano 9.09s,
      nvidia/Cosmos3-Super 8.83s, LiquidAI/LFM3-VL-2B 8.52s,
      LiquidAI/LFM3-Omni-2B 5.57s
  - ruff check and ruff format --check: pass.

Not done (required by AGENTS.md / pr-checklist before any PR)
  - GPU generation tests (tests/models/multimodal/generation/test_qwen3_vl.py).
  - Before/after lm_eval accuracy on a video task (model-affecting change).
  - End-to-end serving reproducer with a real video request.
  - Full pre-commit hooks in the repo's uv/.venv setup (only ruff was run).
  - Human review of every changed line by the submitter.
  - Signed-off-by uses a placeholder identity; amend with your own name and
    email (git commit --amend -s) before any PR, for the DCO check.

AI assistance: written with Claude Code; must be disclosed in any PR.
```
