# Harbor registry + custom spamd image

In-cluster Harbor at `http://harbor.k8s.musikcheck.dk` (MetalLB LoadBalancer via the
chart's nginx service, DNS record created by external-dns in the `k8s.musikcheck.dk`
zone). Plain HTTP on the LAN; kubelet reaches it through a containerd mirror entry.

Layout:

- `apps/08-registry/harbor/app` — HelmRelease (chart `harbor` 1.19.4, helm.goharbor.io),
  Longhorn PVCs, admin password from the `harbor-admin` secret.
- `apps/08-registry/harbor/config` — secrets + the `spamd-image-build` CronJob.
- `images/spamd/Dockerfile` — `instantlinux/spamassassin` + `openssh-client`.
- `clusters/minis/registries/spamassassin-upstream.yaml` — Flux watch on the upstream
  Docker Hub image; its resolved tag is written into the build CronJob's `SPAMD_TAG`
  (setter comment), so a build only starts for a genuinely new upstream release
  (the job exits early if Harbor already has the tag).
- `clusters/minis/registries/spamassassin.yaml` — now watches
  `harbor.k8s.musikcheck.dk/mail/spamd` (`insecure: true`, pull secret `harbor-pull`)
  and drives the spamd workloads in `apps/07-mail/spamassassin`.

## One-time bootstrap steps

1. Encrypt the new secrets before committing (age key from `.sops.yaml`):

   ```sh
   sops encrypt --in-place \
     apps/08-registry/harbor/config/secret-admin.yaml \
     apps/08-registry/harbor/config/secret-robot-push.yaml \
     apps/08-registry/harbor/config/secret-build-git.yaml \
     clusters/minis/registries/harbor-pull-secret.yaml \
     apps/07-mail/spamassassin/secret-harbor-pull.yaml
   ```

   Set the real admin password first: `sops edit apps/08-registry/harbor/config/secret-admin.yaml`
   (keys `HARBOR_ADMIN_PASSWORD` and the `extra-values.yaml` DB password).

2. Commit + push; Flux deploys Harbor. Wait for `harbor-core`, `harbor-registry`,
   `harbor-nginx` to be ready and for the DNS record:
   `dig harbor.k8s.musikcheck.dk` should return a 192.168.8.x address.

3. Configure the k3s nodes to pull over HTTP (Kairos machine config, outside this
   repo — add to the cloud-init `write_files` of every node, then reboot or restart
   k3s so containerd picks it up):

   ```yaml
   # /etc/rancher/k3s/registries.yaml
   mirrors:
     "harbor.k8s.musikcheck.dk":
       endpoint:
         - "http://harbor.k8s.musikcheck.dk"
   ```

   Verify on a node: `sudo crictl pull harbor.k8s.musikcheck.dk/mail/spamd:4.0.2-4`
   (after the first build below).

4. In the Harbor UI (`http://harbor.k8s.musikcheck.dk`, admin password from step 1):
   - create project `mail` (private);
   - robot account `mail-push` with push+pull on `mail` → fill
     `harbor-robot-push` (username/password) in
     `apps/08-registry/harbor/config/secret-robot-push.yaml` via `sops edit`;
   - robot account `mail-pull` with pull-only on `mail` → fill the
     `.dockerconfigjson` in `clusters/minis/registries/harbor-pull-secret.yaml` and
     `apps/07-mail/spamassassin/secret-harbor-pull.yaml` (base64 of `user:pass`);
     both are pre-filled in git already;
   - the build clone uses the same read-only SSH deploy key as Flux
     (`harbor-build-git` secret, keys `identity` + `known_hosts`) — no extra
     GitHub token needed.
   Commit + push.

5. Trigger the first build: `kubectl -n harbor create job --from=cronjob/spamd-image-build spamd-build-$(date +%s)`.
   The image `mail/spamd:4.0.2-4` appears in Harbor; Flux then updates the spamd
   Deployment and both CronJobs to `harbor.k8s.musikcheck.dk/mail/spamd:<tag>`
   (imagepolicy setter comments), and the `default`/`spamd-sa-update` service
   accounts pull it using the `harbor-pull` secret.

6. Verify the backup end-to-end:
   `kubectl -n mail create job --from=cronjob/spamd-bayes-backup test-backup`
   and check `/tmp/root-bayes.txt` on mail2.musikcheck.dk — one container now does
   both the `sa-learn --backup` and the `scp`.

## Follow-ups (done)

- **Flux can push again**: the `flux-fleet-infra` deploy key on
  HasseJohansen/fleet-infra was recreated with "Allow write access"
  (`read_only: false`), so ImageUpdateAutomation commits tag bumps to `./apps`
  again (verified: `apps-image-update` Ready, "repository up-to-date").
- **Node mirror persists**: `write_files` in kairos-config
  (`clusters/minis/cloud-config.yaml`, repo HasseJohansen/kairos-config) writes
  `/etc/rancher/k3s/registries.yaml` on redeploy and every boot, so the
  `harbor.k8s.musikcheck.dk` HTTP mirror survives reboots (applied at runtime on
  all 3 nodes already).
- Upstream tag `4.0.2-4` on Docker Hub is a Helm chart manifest, not an image;
  the upstream ImagePolicy range excludes it via an OR clause
  (`>=4.0.0-0 <4.0.2-4 || >=4.0.3-0 <4.1.0-0`) so any newer real image tag is
  picked up automatically.
- The container `image:` fields in `apps/07-mail/spamassassin/` use the whole-image
  marker `# {"$imagepolicy": "flux-system:spamassassin"}` (no `:tag` suffix): with
  this Flux version the `:tag` marker on a plain image string wrote a bare tag and
  broke pulls. The `:tag` suffix is only used on the plain `SPAMD_TAG` value in the
  build CronJob, where a bare tag is correct.

## Notes

- Upstream tags are prerelease-style (`4.0.2-4`), hence the ImagePolicy ranges use
  `>=4.0.0-0 <4.1.0-0` so they actually match.
- The build CronJob runs hourly but is a no-op unless the desired tag is missing in
  Harbor. The desired tag is `${SPAMD_TAG}.${LOCAL_REV}`: `SPAMD_TAG` tracks upstream
  releases, and you bump `LOCAL_REV` in `spamd-build-cronjob.yaml` whenever
  `images/spamd/Dockerfile` changes (e.g. `4.0.2-3` + `LOCAL_REV=3` → `4.0.2-3.3`).
  Worst-case latency upstream-release → new spamd running: ~30m (upstream
  scan) + ≤1h (cron tick) + build + ~30m (Harbor scan).
- Harbor PVCs are on Longhorn, so the existing Wasabi backup target covers them.
- Set a retention rule in the Harbor UI (e.g. keep the last 10 tags of `mail/spamd`)
  and schedule registry GC.
