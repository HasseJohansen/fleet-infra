# spamd Bayes database

spamd runs in the cluster (apps/07-mail/spamassassin) with the Bayes DB on the shared
RWX PVC `spamd-bayes`, pinned via `bayes_path /var/lib/spamassassin/bayes` in
`local.cf` (`lock_method nfssafe` because the volume is NFS/Longhorn).

- Weekly `spamd-sa-update` CronJob runs `sa-update && sa-compile` on the PVC and then
  deletes the spamd pods so they pick up new rules. The restart container uses
  `bitnami/kubectl:latest` — Bitnami no longer publishes `1.x` version tags for this
  image (only `latest` + digest tags), so the previous `bitnami/kubectl:1.37` tag was
  permanently ImagePullBackOff and rule updates never ran.
- Daily `spamd-bayes-backup` CronJob does `sa-learn --backup` and scp's the dump to
  `mail2.musikcheck.dk:/tmp/root-bayes.txt` (one-way: cluster -> Hetzner host).

## One-time import of the pre-cluster Bayes DB (done 2026-11-23)

Before spamd moved into the cluster it ran on the mail host with a per-user DB in
`/home/hhj/.spamassassin`. That DB was never copied to the PVC (the backup CronJob
only pushes cluster -> mail2), so the cluster Bayes started cold and obvious spam
scored BAYES_00/40/50 instead of BAYES_99. Import procedure (both sides run
SpamAssassin 4.0.2, so the dump format is compatible):

```sh
# on the mail host, as hhj:
sa-learn --sync
sa-learn --backup > /tmp/bayes-backup.txt

# into the cluster PVC (RWX, visible to both spamd pods):
kubectl cp /tmp/bayes-backup.txt mail/<spamd-pod>:/tmp/bayes-backup.txt -c spamd
kubectl exec -n mail <spamd-pod> -c spamd -- \
  sh -c 'sa-learn --restore /tmp/bayes-backup.txt && sa-learn --sync'

# verify: nspam/nham should match the source dump
kubectl exec -n mail <spamd-pod> -c spamd -- sa-learn --dump magic | head
# -> nspam 13816, nham 26322, ntokens 141635 (imported 2026-11-23)
```
