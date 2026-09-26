# The image of robot(): Robot Framework and RequestsLibrary on the
# Python image of python().
#
# The base and both packages are pinned, so the image is the same
# wherever it is built.
FROM python:3.12.14-alpine3.24@sha256:4c47124a8391cb7a9f571164147d154777cf012a4ece5f86097130d7a4478111

ARG ROBOTFRAMEWORK_VERSION=7.5
ARG REQUESTS_LIBRARY_VERSION=0.9.7

RUN pip install --no-cache-dir --disable-pip-version-check \
      "robotframework==${ROBOTFRAMEWORK_VERSION}" \
      "robotframework-requests==${REQUESTS_LIBRARY_VERSION}"

CMD ["robot", "--version"]
