# syntax=docker/dockerfile:1
# instantlinux/spamassassin + openssh-client, so the bayes backup CronJob can do
# both the sa-learn dump and the scp in a single container.
# The build CronJob always passes --build-arg BASE_TAG=<tag resolved by Flux from
# the latest upstream 4.0.x release>; the default below is bumped by Renovate.
ARG BASE_TAG=4.0.2-4
FROM instantlinux/spamassassin:${BASE_TAG}

RUN apt-get update \
    && apt-get install -y --no-install-recommends openssh-client \
    && rm -rf /var/lib/apt/lists/*
