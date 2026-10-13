I am rethinking how we should modernize nx-decomp-tools. I think moving everything to UV is not the final solution. The main pain-point is that essentially everything in nx-decomp-tools assumes the downstream project is setup in the way it wants. The best example is how `viking` calls `asm-differ`:

- `asm-differ` must be cloned at `tools/common/asm-differ`
- `diff_settings.py` must exist at `tools/common/diff_settings.py`
- `config.toml` must exist at `tools/config.toml`
- The current Python interpreter must be have all the necessary dependencies installed.

These assumptions make it hard to package the tool and makes it a pain to set up the project other than using devcontainers or Nix. For someone who wants to set up the project on their regular linux distro (or WSL), would require:

- Installing Python or some kind of python version manager
- Installing required pip packages, either into the system Python and hope nothing conflicts, or bring their own virtualenv.

While devcontainers are great for running scripts, it's not great for IDEs. When editing the scripts I would like the language server to work. We should not require contributors to figure out how to run their IDE inside the devcontainer.

Here's my proposal for rewriting it:

- The downstream-project-facing API will just be a Python package.
- The Python package itself will be built with `uv`+`maturin`+`pyo3`. The rust parts become an extension module.
  - This means we can eliminate the duplicated functionality in python and rust
  - It will probably be Rust-first, with a thin Python API wrapper, with some additional wiring to call asm-differ seamlessly
- All configs, including paths, will be passed-in in a single config file. The tool itself does not assume any project structure other than the location of the config file, which can be discovered based on things like environment variable or from the current directory using a recursive-parent strategy (for parents of current directory, check if `tools/config.toml` exists). Tool resolution follows a similar approach (environment variable override > location defined in config file > system PATH)
- The package will publish source on GitHub and prebuilt wheel on GitHub Release. Installing it from prebuilt wheel means a Rust toolchain is not required. It does not care how downstream projects install or call it:
  - Install with `pip`:
    - prebuilt: `pip install https://github.com/open-ead/nx-decomp-tools/releases/download/v1.2.3/xxx.whl`
    - source: `pip install git+https://github.com/open-ead/nx-decomp-tools.git@v1.2.3`
  - Install with `uv`:
    - prebuilt: (in pyproject.toml) `mypackage = { url = "https://github.com/OWNER/REPO/releases/download/v1.2.3/xxx.whl" }`
    - source: `uv add "mypackage @ git+https://github.com/OWNER/REPO.git@v1.2.3"`
  - Calling a tool (like check):
    - From python: `python -m nx_decomp_tools check`
    - From uv: `uv run nx_decomp_tools check`
  - Importing from a script (for things like setup):
    - `from nx_decomp_tools.xxx import yyy`

Other thoughts:

- What about ida_scripts and ghidra_scripts?? - They will probably be replaced by BLSS anyway
- Name? `nx_decomp_tools` is kind of chunky, could we just call everything `viking`?
