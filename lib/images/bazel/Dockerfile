# The image of bazel(): Bazelisk, which runs the Bazel release a
# project names in .bazelversion, on Debian with the toolchains most
# rules expect (a C++ compiler, Python, Git). Bazel's own image is
# published for amd64 only.
#
# The base is pinned by its digest, and Bazelisk by its version and
# checksum.
FROM debian:trixie-slim@sha256:a99cfc517144bc59b1978475ec53b46ecabec7e43635402ee5b77cc54cd1b20a

ARG BAZELISK_VERSION=1.29.0

RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl git python3 g++ unzip zip \
 && rm -rf /var/lib/apt/lists/*

RUN arch="$(dpkg --print-architecture)" \
 && case "$arch" in \
      amd64) sum=5a408715e932c0250d28bd84555f12edbf70117de42f9181691c736eacc4a992 ;; \
      arm64) sum=e20e8b0f4f240091b7a55bf17b9398bd4f40ee70ae0208dff95dd4c445fb4010 ;; \
      *) echo "unsupported architecture: $arch" >&2; exit 1 ;; \
    esac \
 && curl -fsSL -o /usr/local/bin/bazelisk "https://github.com/bazelbuild/bazelisk/releases/download/v${BAZELISK_VERSION}/bazelisk-linux-${arch}" \
 && echo "${sum}  /usr/local/bin/bazelisk" | sha256sum -c - \
 && chmod +x /usr/local/bin/bazelisk \
 && ln -s bazelisk /usr/local/bin/bazel

CMD ["bazel", "--version"]
