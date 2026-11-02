# Appendix B: Code Templates and Implementations

## B.0 Running the Code in This Book

Every snippet in this book runs the same way, on every machine, through one tool: uv. This section is the only place the book explains it; the chapters just point here.

**What uv is.** uv is a single program that does the jobs Python has historically spread across four or five tools. It installs Python itself, so you need neither a system Python nor a version manager (the job pyenv used to do). It resolves and installs packages (pip's job). It gives each script its own isolated environment—a private set of installed libraries—so two scripts with different needs never collide (virtualenv's job), and it runs command-line tools without installing them into anything (pipx's job). You never create, activate, or clean up an environment; uv builds one for each command, caches what it downloaded, and throws the environment away when the command finishes. That is why the book standardizes on it: there is no setup step a snippet can fail on because you skipped it. Paste, run, done.

**Installing uv.** One line. On macOS or Linux:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

On Windows, in PowerShell (the backtick is PowerShell's line-continuation character, so these two lines are one command):

```powershell
powershell -ExecutionPolicy ByPass -c `
  "irm https://astral.sh/uv/install.ps1 | iex"
```

If you already use Homebrew:

```bash
brew install uv
```

Open a new terminal afterward and `uv --version` should answer. That is the whole installation.

**Running a one-off snippet.** Most code in this book is a few lines meant to be read and tried, not saved. Hand it to `python -c` and tell uv which libraries it imports, one `--with` per library:

```bash
uv run --with pandas --with scikit-learn python -c \
  "from sklearn.datasets import load_iris; \
   print(load_iris(as_frame=True).frame.describe())"
```

The first time, uv downloads pandas, scikit-learn, and everything they depend on, which takes a few seconds—and it downloads Python too if your machine has none. The second time, the same command starts in well under a second, because everything is already in uv's cache. The name after `--with` is the package's name on the Python Package Index, which is usually, but not always, the name you import; the table below covers the exceptions.

**Saving a snippet as a script.** When a snippet is worth keeping, put it in a file that opens with a three-line header declaring what it needs:

```python
# /// script
# dependencies = ["pandas", "scikit-learn"]
# ///
```

Your code follows the header. The header is a Python standard (PEP 723, inline script metadata), not a uv invention, and uv reads it, so running the file needs nothing else:

```bash
uv run snippet.py
```

No `--with` flags, no environment, no requirements file. The script carries its own shopping list. Every script in this book that you are meant to save has this header.

**Command-line tools.** Some of what the book uses is a program rather than a library—DVC in Chapter 5, for example. `uvx` runs a tool in a throwaway environment of its own:

```bash
uvx dvc add data/train.parquet
```

Read `uvx dvc` as "run dvc, fetching it if necessary." Nothing is installed globally and nothing is left behind.

**Notebooks.** If you would rather work in a notebook, uv runs Jupyter the same way:

```bash
uv run --with jupyter jupyter lab
```

Add a `--with` for each library you plan to import inside the notebook; the kernel sees exactly what you asked for and nothing else.

**Import name versus package name.** A handful of libraries install under one name and import under another. `--with` and the `dependencies` list take the package name; your `import` line takes the import name.

| You write `import ...` | You tell uv `--with ...` |
|---|---|
| `sklearn` | `scikit-learn` |
| `cv2` | `opencv-python` |
| `PIL` | `pillow` |
| `yaml` | `pyyaml` |
| `lance` | `pylance` |

Everything else in this book—pandas, numpy, scipy, pyarrow, polars, duckdb, and the rest—imports under the same name it installs under.

**Why there are no version numbers in this book.** You will not find a library version number anywhere in these pages, and the `dependencies` lists carry no pins. uv resolves the current release of every library each time it builds an environment, so the code targets each library as it exists today, not as it existed when a sentence was written. The honest consequence is that a snippet can stop working when a library ships a release that changes something. Treat that as a signal, not a nuisance: the library's changelog is the first place to look, and the fix is nearly always a renamed argument or a moved import. A book that pinned versions would hide that signal from you until the day the pinned versions stopped installing at all.

*Sections B.1 to B.5 from the Outline follow later.*
