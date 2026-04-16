# ComfyUI My Nodes

[![GitHub Stars](https://img.shields.io/github/stars/YOUR_USERNAME/ComfyUI-MyNodes?style=flat)](https://github.com/YOUR_USERNAME/ComfyUI-MyNodes/stargazers)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/)

Short one-line description of what your nodes do.

---

## Nodes

| Node | Description |
|---|---|
| **Load My Model** | Downloads and loads the model with automatic precision detection. |
| **Example Image Node** | Processes an image using the loaded model. |
| **Save My Image** | Saves output images to the ComfyUI output folder. |

---

## Installation

### ComfyUI Manager (recommended)

1. Open **ComfyUI Manager**
2. Search for `My Nodes`
3. Click **Install** → restart ComfyUI

### Manual

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/YOUR_USERNAME/ComfyUI-MyNodes
```

Restart ComfyUI. Dependencies install automatically on first run.

---

## Requirements

- ComfyUI (latest)
- GPU with 8 GB+ VRAM recommended
- Model downloads automatically from HuggingFace on first use (~X GB)

---

## Usage

1. Add **Load My Model** → connect `model_config` to **Example Image Node**
2. Connect an image source
3. Optionally attach **Save My Image** to write results to disk

Example workflows are in the [`workflows/`](workflows/) folder.

---

## Support

- Open a [GitHub Discussion](https://github.com/YOUR_USERNAME/ComfyUI-MyNodes/discussions) for questions
- File bugs via [GitHub Issues](https://github.com/YOUR_USERNAME/ComfyUI-MyNodes/issues)

---

## Credits

- [Model Name](https://arxiv.org/abs/XXXX.XXXXX) — Paper title (Conference Year)
