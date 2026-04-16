# ComfyUI Custom Node Template — General Plan

A clean, general-purpose template for building ComfyUI custom node packages using the **V3 API**.

---

## File Tree

```
ComfyUI-MyNodes/
├── __init__.py                  ← ComfyExtension entrypoint (V3)
├── install.py                   ← comfy-env: install isolated deps
├── prestartup_script.py         ← comfy-env: setup env + copy assets to ComfyUI/input
├── pyproject.toml               ← Package metadata + [tool.comfy] registry block
├── requirements.txt             ← Pip dependencies (exclude ComfyUI built-ins)
├── comfy-env-root.toml          ← Declare other ComfyUI node deps + brew packages
├── comfy-test.toml              ← Test matrix config for comfy-test CI
├── LICENSE                      ← MIT
├── README.md                    ← Badges, nodes table, install instructions, credits
├── nodes/
│   ├── __init__.py              ← NODE_CLASSES list — import and list all node classes here
│   ├── load_model.py            ← HF Hub auto-download + precision/dtype resolution
│   ├── example_node.py          ← Image-in → model → image-out template
│   ├── output_node.py           ← Save image to output folder + preview
│   └── utils.py                 ← Shared helpers (tensor↔PIL, resize, etc.)
├── js/
│   └── extension.js             ← Frontend JS hooks (setup, beforeRegisterNodeDef, etc.)
├── workflows/                   ← Example .json workflows for users + CI testing
├── assets/                      ← Static files copied to ComfyUI/input at startup
├── docs/                        ← Per-node markdown help (filename = node_id)
└── .github/
    └── workflows/
        ├── publish.yml          ← Publishes to ComfyUI Registry on version tag push
        ├── pr-gate.yml          ← Runs comfy-test matrix on PR / push to main
        └── run-tests.yml        ← Runs comfy-test matrix on push to dev branch
```

### CI: comfy-test (PozzettiAndrea/comfy-test)

`pr-gate.yml` and `run-tests.yml` both delegate to the shared reusable workflow:

```yaml
jobs:
  test:
    uses: PozzettiAndrea/comfy-test/.github/workflows/test-matrix.yml@main
    with:
      config-file: comfy-test.toml
```

**7 progressive test levels** (configured in `comfy-test.toml`):
`syntax` → `install` → `registration` → `instantiation` → `static_capture` → `validation` → `execution`

Put test workflows (`.json`) in the `workflows/` folder — comfy-test picks them up automatically.

### comfy-env (PozzettiAndrea/comfy-env)

For nodes with heavy or conflicting deps, use process isolation:
- `install.py` — runs `comfy_env.install()` on first install
- `prestartup_script.py` — runs `setup_env()` before ComfyUI starts
- `comfy-env-root.toml` — declares other node deps and brew packages
- For full subprocess isolation, add a `comfy-env.toml` in a subdirectory

---

## Registration (`__init__.py`)

V3 style — the recommended approach for all new packages:

```python
from typing_extensions import override
from comfy_api.latest import ComfyExtension, io

WEB_DIRECTORY = "./js"

class MyExtension(ComfyExtension):
    @override
    async def get_node_list(self) -> list[type[io.ComfyNode]]:
        from .nodes import NODE_CLASSES
        return NODE_CLASSES

    @override
    async def on_load(self):
        pass  # optional init (e.g. register NodeReplace migrations)

async def comfy_entrypoint() -> MyExtension:
    return MyExtension()
```

> **V1 alternative** (wider compat, no async):
> ```python
> from .nodes import NODE_CLASSES
> NODE_CLASS_MAPPINGS = {cls.define_schema().node_id: cls for cls in NODE_CLASSES}
> NODE_DISPLAY_NAME_MAPPINGS = {cls.define_schema().node_id: cls.define_schema().display_name for cls in NODE_CLASSES}
> WEB_DIRECTORY = "./js"
> ```

---

## Node Class Pattern

Every node inherits `io.ComfyNode` and implements `define_schema` + `execute`:

```python
from comfy_api.latest import io

class MyNode(io.ComfyNode):
    @classmethod
    def define_schema(cls):
        return io.Schema(
            node_id="MyNode",                   # globally unique — never change after release
            display_name="My Node",
            category="MyNodes",
            description="One-line tooltip.",
            inputs=[
                io.Image.Input("image"),
                io.Float.Input("strength", default=1.0, min=0.0, max=2.0, step=0.01,
                               display_mode=io.NumberDisplay.slider),
                io.Int.Input("seed", default=0, min=0, max=0xFFFFFFFFFFFFFFFF,
                             control_after_generate=True),
                io.Combo.Input("mode", options=["fast", "quality"], default="fast"),
                io.Boolean.Input("enabled", default=True),
                io.Mask.Input("mask", optional=True),
            ],
            outputs=[
                io.Image.Output("IMAGE"),
            ],
            hidden=[io.Hidden.unique_id],       # access via cls.hidden.unique_id in execute
        )

    @classmethod
    def validate_inputs(cls, image, strength, seed, mode, enabled, mask=None):
        # Return True if ok, or an error string to block execution.
        return True

    @classmethod
    def fingerprint_inputs(cls, image, strength, seed, mode, enabled, mask=None):
        # Return a value; if different from last run → re-execute.
        # Omit this method to cache based on input values (default).
        return strength

    @classmethod
    def execute(cls, image, strength, seed, mode, enabled, mask=None):
        import torch
        result = torch.clamp(image * strength, 0.0, 1.0)
        return io.NodeOutput(result)
```

---

## Custom Types Between Nodes

Pass opaque config objects (e.g. loaded model handles) between nodes:

```python
# In load_model.py — define once, import everywhere
MY_MODEL_CONFIG = io.Custom("MY_MODEL_CONFIG")

# Output from loader:
outputs=[MY_MODEL_CONFIG.Output(display_name="model_config")]

# Input to processing node:
inputs=[MY_MODEL_CONFIG.Input("model_config")]
```

---

## Model Loading Pattern

```python
import folder_paths, torch, comfy.model_management
from pathlib import Path
from huggingface_hub import hf_hub_download

checkpoint = Path(folder_paths.models_dir) / "your_model" / "model.safetensors"

if not checkpoint.exists():
    hf_hub_download(repo_id="org/repo", filename="model.safetensors",
                    local_dir=str(checkpoint.parent))

device = comfy.model_management.get_torch_device()

# Auto precision
if comfy.model_management.should_use_bf16(device):   dtype = torch.bfloat16
elif comfy.model_management.should_use_fp16(device): dtype = torch.float16
else:                                                  dtype = torch.float32

# Store dtype as string for JSON-safe IPC across subprocess boundaries
dtype_str = {torch.bfloat16: "bf16", torch.float16: "fp16", torch.float32: "fp32"}[dtype]
```

---

## Output Node (save + preview)

```python
from comfy_api.latest import io, ui

class SaveMyImage(io.ComfyNode):
    @classmethod
    def define_schema(cls):
        return io.Schema(
            node_id="SaveMyImage",
            display_name="Save My Image",
            category="MyNodes",
            is_output_node=True,
            inputs=[
                io.Image.Input("images"),
                io.String.Input("filename_prefix", default="ComfyUI"),
            ],
            outputs=[],
            hidden=[io.Hidden.prompt, io.Hidden.extra_pnginfo],
        )

    @classmethod
    def execute(cls, images, filename_prefix):
        saved = ui.ImageSaveHelper.get_save_images_ui(images, filename_prefix, cls=cls)
        return io.NodeOutput(ui=saved)
```

---

## Common Input Types Quick Reference

| Need | Code |
|---|---|
| Image tensor | `io.Image.Input("image")` |
| Mask | `io.Mask.Input("mask", optional=True)` |
| Float slider | `io.Float.Input("v", default=1.0, min=0.0, max=2.0, display_mode=io.NumberDisplay.slider)` |
| Integer | `io.Int.Input("steps", default=20, min=1, max=150)` |
| Seed | `io.Int.Input("seed", default=0, max=0xFFFFFFFFFFFFFFFF, control_after_generate=True)` |
| Dropdown | `io.Combo.Input("mode", options=["a","b","c"], default="a")` |
| Checkbox | `io.Boolean.Input("flag", default=True)` |
| Multiline text | `io.String.Input("prompt", multiline=True)` |
| Custom type | `MY_TYPE = io.Custom("MY_TYPE")` → `MY_TYPE.Input("x")` |
| Any type | `io.AnyType.Input("anything")` |
| Multiple types | `io.MultiType.Input("data", types=[io.Image, io.Mask])` |

---

## Advanced Patterns (import from skills when needed)

| Pattern | Skill |
|---|---|
| Dynamic growing inputs | `comfyui-node-advanced` → `io.Autogrow` |
| Generic type propagation | `comfyui-node-advanced` → `io.MatchType` |
| Conditional sub-inputs | `comfyui-node-advanced` → `io.DynamicCombo` |
| Subgraph injection | `comfyui-node-advanced` → `GraphBuilder` + `enable_expand=True` |
| Lazy evaluation | `comfyui-node-inputs` → `lazy=True` + `check_lazy_status` |
| Progress reporting | `comfyui-node-advanced` → `ComfyAPISync().execution.set_progress(i, n)` |
| Node migration | `comfyui-node-advanced` → `io.NodeReplace` in `on_load` |
| Frontend widgets/tabs | `comfyui-node-frontend` → `app.registerExtension({...})` |
| Async execute | `comfyui-node-advanced` → `async def execute(cls, ...)` + `ComfyAPI` |

---

## pyproject.toml

```toml
[project]
name = "comfyui-my-nodes"
version = "0.1.0"
description = "My custom nodes for ComfyUI."
license = { file = "LICENSE" }
requires-python = ">=3.10"

[project.urls]
Repository = "https://github.com/YOUR_USERNAME/ComfyUI-MyNodes"

[tool.comfy]
PublisherId = "your_publisher_id"
DisplayName = "My Nodes"
Icon = ""
```

---

## Checklist When Starting a New Package

- [ ] Replace `MY_MODEL_CONFIG`, `MyExtension`, `MyNode`, `MyNodes` with your names
- [ ] Set `HF_REPO_ID` / `HF_FILENAME` / `MODEL_SUBDIR` in `load_model.py`
- [ ] Update `pyproject.toml` — name, description, publisher ID, repo URL
- [ ] Update `publish.yml` — replace `YOUR_USERNAME`
- [ ] Fill in `requirements.txt`
- [ ] Update `LICENSE` — replace `YOUR_NAME`
- [ ] Change `node_id` strings (must be globally unique — never change after first release)
