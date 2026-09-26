# The image of protoc(): the Protocol Buffers compiler with the
# well-known types (/usr/include/google/protobuf) and the gRPC plugins
# (grpc_cpp_plugin, grpc_python_plugin and the rest).
#
# The base is pinned by its digest; the packages are Alpine 3.24's.
FROM alpine:3.24@sha256:294b683cb724975bec92580e1e685676bd4b50bda910ddb8c51d4cabeaec77e6

RUN apk add --no-cache protobuf protobuf-dev grpc-plugins

CMD ["protoc", "--version"]
