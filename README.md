# l6ds-base

Base repository for Level 6 Data Science modules. It provides two things:

1. **A template repository.** Repos created from it open in a ready-to-use VS Code devcontainer.
2. **Two published container images** built from `.devcontainer/Dockerfile`:

| Image | Dockerfile target | What it is |
| --- | --- | --- |
| `ghcr.io/bpp-sot/l6ds-base` | `base` | Python 3.12 (Debian bookworm), the data-science stack in `requirements.txt`, and common NLTK data. Used by the devcontainer. |
| `ghcr.io/bpp-sot/l6ds-vscode` | `vscode` | `l6ds-base` plus [code-server](https://github.com/coder/code-server), which serves VS Code in the browser on port 8080. Intended for hosted labs such as Skillable. |

## What's installed

- **Data:** numpy, pandas, scipy, pyarrow, openpyxl
- **ML/stats:** scikit-learn, statsmodels, xgboost, imbalanced-learn
- **NLP:**
  - nltk, with common data baked in at `/usr/local/share/nltk_data` so it works offline: `punkt`, `punkt_tab`, `stopwords`, `wordnet`, `omw-1.4`, `vader_lexicon`, the POS taggers, `maxent_ne_chunker_tab` and `words`
  - TextBlob, including its corpora (`brown`, `conll2000`, `movie_reviews`)
  - spaCy, with the `en_core_web_sm` English model
- **Visualisation:** matplotlib, seaborn, plotly
- **Notebooks:** jupyterlab, ipykernel, ipywidgets
- **Utilities:** requests, beautifulsoup4, tqdm
- **Browser automation:** selenium, with Debian's `chromium` and matching `chromium-driver` (see below)
- **VS Code extensions:** Python, Jupyter, Black formatter, Rainbow CSV

Versions are pinned in `requirements.txt`.

## Selenium with headless Chromium

Chromium and chromedriver are installed from Debian, so their versions always match. `SE_OFFLINE=true` stops Selenium trying to download its own driver. Containers need these Chrome flags:

```python
from selenium import webdriver

options = webdriver.ChromeOptions()
options.add_argument("--headless=new")
options.add_argument("--no-sandbox")             # required inside containers
options.add_argument("--disable-dev-shm-usage")  # /dev/shm is small in Docker

driver = webdriver.Chrome(options=options)
driver.get("https://example.com")
print(driver.title)
driver.quit()
```

## Using the template

Create a repo from this template, open it in VS Code and choose **Reopen in Container**. The devcontainer pulls the prebuilt `l6ds-base` image. Add any extra packages for your project to `requirements.txt`; they are installed when the container is created.

## Using the browser VS Code image (Skillable)

```bash
docker run -p 8080:8080 ghcr.io/bpp-sot/l6ds-vscode:latest
```

Then open <http://localhost:8080>. The workspace folder is `/home/vscode/workspace`, and the editor runs as the non-root `vscode` user.

| Variable | Default | Purpose |
| --- | --- | --- |
| `CODE_SERVER_AUTH` | `none` | `none`, or `password` to require a login |
| `PASSWORD` | – | Login password when `CODE_SERVER_AUTH=password` |
| `CODE_SERVER_PORT` | `8080` | Port code-server listens on |

Notes:

- Extensions come from Open VSX. Pylance is not available there, so Python IntelliSense uses Jedi.
- Authentication defaults to `none`, on the assumption that the lab platform controls access. Set a password if the port is exposed anywhere else.

## Building and publishing

GitHub Actions (`.github/workflows/publish-images.yml`) builds both images and pushes them to GHCR. It runs:

- on pushes to `main` that change the Dockerfile, `requirements.txt` or the workflow
- on `v*` tags, which also produce semver image tags
- manually, with `workflow_dispatch`

The workflow only runs in `bpp-sot/l6ds-base`, not in repos created from the template.

Images are always pushed to GHCR. They're also pushed to Docker Hub (as `<namespace>/l6ds-base` and `<namespace>/l6ds-vscode`), which Skillable needs, when these repository settings exist:

| Setting | Kind | Value |
| --- | --- | --- |
| `DOCKERHUB_NAMESPACE` | Actions **variable** | Docker Hub user or organisation to publish under |
| `DOCKERHUB_USERNAME` | Actions **secret** | Docker Hub user that owns the access token |
| `DOCKERHUB_TOKEN` | Actions **secret** | Docker Hub personal or organisation access token with *Read & Write* access |

To build locally:

```bash
docker build -f .devcontainer/Dockerfile --target base   -t l6ds-base .
docker build -f .devcontainer/Dockerfile --target vscode -t l6ds-vscode .
```

To update a library, change its pin in `requirements.txt` and push. To bump code-server, change `CODE_SERVER_VERSION` in the Dockerfile.
