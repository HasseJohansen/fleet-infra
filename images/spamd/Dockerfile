# syntax=docker/dockerfile:1
# instantlinux/spamassassin + openssh-client, so the spamd bayes backup CronJob can
# do both the sa-learn dump and the scp in a single container.
# BASE_TAG is passed by the spamd-image-build CronJob as --build-arg, resolved by
# Flux from the latest upstream 4.0.x release; the default is bumped by Renovate.
ARG BASE_TAG=4.0.2-4
FROM instantlinux/spamassassin:${BASE_TAG}

# PATH is set explicitly: the base image config carries no PATH env, so a bare
# "apt-get" is not found by kaniko's /bin/sh -c.
RUN PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
    apt-get update \
    && apt-get install -y --no-install-recommends openssh-client \
    && rm -rf /var/lib/apt/lists/*
