# ComfyUI Custom Node Template

Template repository for building ComfyUI custom node packages with the V3 API.

Use [README.md.example](/home/wswg3/project/comfyui-template/README.md.example) as the starting point for the actual project README in repositories created from this template.

## What This Template Includes

- V3 `ComfyExtension` entrypoint in [`__init__.py`](/home/wswg3/project/comfyui-template/__init__.py)
- example loader, processing, and output nodes in [`nodes/`](/home/wswg3/project/comfyui-template/nodes)
- optional `comfy_env` integration with a safe `pip` fallback in [`install.py`](/home/wswg3/project/comfyui-template/install.py) and [`prestartup_script.py`](/home/wswg3/project/comfyui-template/prestartup_script.py)
- package metadata in [`pyproject.toml`](/home/wswg3/project/comfyui-template/pyproject.toml)
- registry publish workflow in [publish.yml](/home/wswg3/project/comfyui-template/.github/workflows/publish.yml)

## How To Use This Repo

1. Copy this template into a new repository.
2. Update placeholders in `pyproject.toml`, the node files, and workflow metadata.
3. Use `README.md.example` as the base for your real project `README.md`.
4. Replace the example node logic with your actual model or processing code.

## Notes

- `README.md` describes the template repo itself.
- `README.md.example` is the end-user README template for future projects created from this repo.

## License

This repository is licensed under the MIT License. See [LICENSE](/home/wswg3/project/comfyui-template/LICENSE).

## Acknowledgements

- PozzettiAndrea's ComfyUI tooling and workflow conventions were used as references for parts of this template, especially around `comfy_env`, `comfy-test`, and maintainer guidance structure.
- [jtydhr88/comfyui-custom-node-skills](https://github.com/jtydhr88/comfyui-custom-node-skills) was also used as a reference.
