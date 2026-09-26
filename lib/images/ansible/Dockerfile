# The image of ansible(): ansible-core on the Python image python()
# runs, with the SSH client a playbook connects through.
#
# Both the base and ansible-core are pinned, so the image is the same
# wherever it is built; lask tags it by the hash of this file.
FROM python:3.12.14-alpine3.24@sha256:4c47124a8391cb7a9f571164147d154777cf012a4ece5f86097130d7a4478111

ARG ANSIBLE_CORE_VERSION=2.21.4

RUN apk add --no-cache openssh-client sshpass git \
 && pip install --no-cache-dir --disable-pip-version-check "ansible-core==${ANSIBLE_CORE_VERSION}"

CMD ["ansible", "--version"]
