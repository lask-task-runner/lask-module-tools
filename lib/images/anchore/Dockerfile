# The image of grype() and syft(): Anchore's scanner and SBOM
# generator on Alpine. Their own images have no /bin/sh, which a lask
# command runs through, so the binaries are copied out of them.
#
# Every image is pinned by its digest.
FROM anchore/grype:v0.119.0@sha256:8c2c9234a345577a6d321a4753aa3ee1276d8975c8452d2344a56b57733ecad3 AS grype
FROM anchore/syft:v1.52.0@sha256:500e2d872ac019436926e8322b4fc1f39441d94d21f6f4046c6ff29b30e8cb02 AS syft

FROM alpine:3.24@sha256:294b683cb724975bec92580e1e685676bd4b50bda910ddb8c51d4cabeaec77e6
RUN apk add --no-cache ca-certificates
COPY --from=grype /grype /usr/local/bin/grype
COPY --from=syft /syft /usr/local/bin/syft

CMD ["grype", "version"]
