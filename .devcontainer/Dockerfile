# syntax=docker/dockerfile:1
# check=skip=SecretsUsedInArgOrEnv
#
# L6 Data Science images
#
#   target "base"   -> ghcr.io/bpp-sot/l6ds-base
#       Python 3.12 + the data-science stack + NLTK data.
#       Used as the VS Code devcontainer for repos created from this template.
#
#   target "vscode" -> ghcr.io/bpp-sot/l6ds-vscode
#       base + code-server (VS Code in the browser) with extensions pre-installed.
#       Intended for hosted labs (e.g. Skillable) where the learner reaches
#       VS Code through a web port rather than a local VS Code install.

# ---------------------------------------------------------------------------
FROM mcr.microsoft.com/devcontainers/python:3.12-bookworm AS base

LABEL org.opencontainers.image.source="https://github.com/bpp-sot/l6ds-base" \
      org.opencontainers.image.description="Python 3.12 data-science environment for Level 6 Data Science modules" \
      org.opencontainers.image.licenses="MIT"

ENV PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    NLTK_DATA=/usr/local/share/nltk_data \
    MPLBACKEND=Agg \
    SE_OFFLINE=true

# Headless Chromium + matching chromedriver for Selenium (from Debian, so the
# browser and driver versions always match). SE_OFFLINE stops Selenium Manager
# trying to download its own browser/driver.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        chromium chromium-driver fonts-liberation \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt /tmp/l6ds/requirements.txt
RUN pip install --no-cache-dir -r /tmp/l6ds/requirements.txt \
    && rm -rf /tmp/l6ds

# Bake commonly-used NLTK data (including the TextBlob corpora) into the image so it works in labs with no
# outbound network access.
RUN python -m nltk.downloader -d "${NLTK_DATA}" \
        punkt punkt_tab stopwords wordnet omw-1.4 vader_lexicon \
        averaged_perceptron_tagger_eng maxent_ne_chunker_tab words \
        brown averaged_perceptron_tagger conll2000 movie_reviews \
    && chmod -R a+rX "${NLTK_DATA}"

# ---------------------------------------------------------------------------
FROM base AS vscode

ARG CODE_SERVER_VERSION=4.143.1
ARG TARGETARCH

RUN curl -fsSL -o /tmp/code-server.deb \
        "https://github.com/coder/code-server/releases/download/v${CODE_SERVER_VERSION}/code-server_${CODE_SERVER_VERSION}_${TARGETARCH:-amd64}.deb" \
    && apt-get install -y --no-install-recommends /tmp/code-server.deb \
    && rm -f /tmp/code-server.deb \
    && rm -rf /var/lib/apt/lists/*

USER vscode
WORKDIR /home/vscode

# Extensions come from Open VSX (code-server's marketplace). Pylance is not
# available there, so the Python extension falls back to Jedi for IntelliSense.
RUN for ext in \
        ms-python.python \
        ms-toolsai.jupyter \
        ms-python.black-formatter \
        mechatroner.rainbow-csv; \
    do code-server --install-extension "$ext"; done \
    && rm -rf /home/vscode/.local/share/code-server/CachedExtensionVSIXs

COPY --chown=vscode:vscode .devcontainer/code-server/settings.json \
     /home/vscode/.local/share/code-server/User/settings.json

RUN mkdir -p /home/vscode/workspace

# Authentication: "none" by default because Skillable fronts the lab with its
# own access control. To require a password instead, run with
#   -e CODE_SERVER_AUTH=password -e PASSWORD=<secret>
ENV CODE_SERVER_AUTH=none \
    CODE_SERVER_PORT=8080

EXPOSE 8080

CMD ["sh", "-c", "exec code-server --bind-addr 0.0.0.0:${CODE_SERVER_PORT} --auth ${CODE_SERVER_AUTH} --disable-telemetry --disable-update-check /home/vscode/workspace"]
