# syntax=docker/dockerfile:1.7
#
# Acof on the Launcher desktop base.
#
# Acof is a single self-contained Go binary: an autonomous security agent for
# vulnerability research, bug hunting and source-code auditing. The agentic
# engine runs in the binary and talks straight to a model provider. It is not an
# interactive TUI - each run takes a task and an authorization scope - so the
# product surface is a persistent terminal in /workspace where the researcher
# drives `acof`, not an agent opened on start.
#
# Build from the repository root so the compile stage sees the Go module:
#
#     docker build -f image/desktop/Dockerfile -t acof-desktop .
#
# The binary is compiled from this repo rather than downloaded from a release,
# so the image can never drift from the source it ships beside.

# The Launcher desktop substrate, published by the Launcher repo. Declared here
# in the global scope (before the first FROM) so it can be used in the product
# stage's FROM below; an ARG consumed by a FROM must precede every FROM. Pin to
# a released substrate version for reproducible builds.
ARG DESKTOP_IMAGE=ghcr.io/pdparchitect/launcher-image-base-desktop:latest

# --- build stage: compile acof from source ---------------------------------
# Pinned to the build platform and cross-compiled with GOOS/GOARCH, so a
# multi-arch build never pays for QEMU on the Go compile.
FROM --platform=${BUILDPLATFORM} golang:1.26.5-bookworm AS build

WORKDIR /src

# Warm the module cache on the manifest alone, so source edits do not re-download
# dependencies. The committed go.mod pins a published SDK, so this needs no
# workspace or local checkout.
COPY go.mod go.sum ./
RUN go mod download

# embed.go bakes skills/ into the binary, so the whole module is the build input.
COPY . .

ARG TARGETOS
ARG TARGETARCH
RUN CGO_ENABLED=0 GOOS="${TARGETOS}" GOARCH="${TARGETARCH}" go build \
        -trimpath \
        -ldflags "-s -w -X github.com/pdparchitect/acof/internal/version.Version=$(cat VERSION)" \
        -o /out/acof ./cmd/acof

# --- product stage: acof on the desktop substrate --------------------------
# DESKTOP_IMAGE is declared in the global scope above so it resolves here.
FROM ${DESKTOP_IMAGE}

USER root

COPY --from=build /out/acof /usr/local/bin/acof
RUN chmod 0755 /usr/local/bin/acof && acof version

# Acof is stateless - it needs only a provider endpoint and a key. What is worth
# persisting is its config: ~/.config/acof/config.yaml, where `acof config`
# stores the provider key (so a researcher does not re-enter it after every
# container replacement). acof reads it directly; it lives under the XDG config
# dir, kept out of the workspace files view.
RUN mkdir -p /home/agent/.config/acof && \
    chmod 0700 /home/agent/.config/acof && \
    chown -R agent:agent /home/agent

# Product entry points, terminal palette, wallpaper, browser page, key handling.
COPY image/desktop/overlay /
RUN chmod 0755 \
        /usr/local/bin/desktop-harness \
        /usr/local/bin/desktop-panel-status \
        /usr/local/bin/desktop-selftest \
        /usr/local/bin/desktop-welcome \
        /usr/local/bin/acof-greeting

# Recolour the inherited desktop chrome into Acof's palette: an obsidian canvas
# and chrome with a single crimson accent - the red-team signature - carrying
# the interactive bits. The script maps the base's colour vocabulary role by
# role rather than replacing themerc, tint2rc and gtk.css, so later substrate
# changes still reach this product. The focused window keeps the base's white
# outline untouched.
COPY image/desktop/theme/apply-theme.sh /tmp/apply-theme.sh
RUN bash /tmp/apply-theme.sh && rm /tmp/apply-theme.sh

RUN kasm-patch "Acof"

ENV DESKTOP_TITLE="Acof" \
    DESKTOP_PERSISTENT_PATHS="/home/agent/.config/acof"

LABEL org.opencontainers.image.title="Acof" \
    org.opencontainers.image.description="Acof autonomous security agent in a Launcher-managed browser desktop" \
    org.opencontainers.image.source="https://github.com/pdparchitect/acof"

VOLUME ["/workspace", "/home/agent/.config/acof"]

WORKDIR /workspace
