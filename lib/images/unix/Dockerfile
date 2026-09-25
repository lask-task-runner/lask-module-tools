# The image of unix(): the everyday command-line tools, in one image so
# that a pipeline such as `curl ... | jq ...` runs in one container.
#
# The base is pinned by its digest. The packages are Alpine 3.24's, as
# the repository has them when the image is built; lask tags the image
# by the hash of this file, so a machine builds it once and keeps it.
FROM alpine:3.24@sha256:294b683cb724975bec92580e1e685676bd4b50bda910ddb8c51d4cabeaec77e6

RUN apk add --no-cache \
      bash coreutils findutils grep sed gawk diffutils file less tzdata \
      ca-certificates curl wget openssl \
      jq yq-go gettext-envsubst \
      tar gzip xz bzip2 zip unzip

CMD ["bash"]
