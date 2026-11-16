# syntax=docker/dockerfile:1
# instantlinux/spamassassin + openssh-client, so the spamd bayes backup CronJob can
# do both the sa-learn dump and the scp in a single container.
# BASE_TAG is passed by the spamd-image-build CronJob as --build-arg, resolved by
# Flux from the latest upstream 4.0.x release; the default is bumped by Renovate.
ARG BASE_TAG=4.0.2-3
FROM instantlinux/spamassassin:${BASE_TAG}

# The base image sets USER 2022 (debian-spamd); kaniko runs RUN as that user, so
# switch to root for the package install and restore the original user afterwards.
# Absolute paths: kaniko's /bin/sh -c does not have /usr/bin in PATH for this base.
USER root
RUN /usr/bin/apt-get update \
    && /usr/bin/apt-get install -y --no-install-recommends openssh-client \
    && rm -rf /var/lib/apt/lists/*
USER 2022
