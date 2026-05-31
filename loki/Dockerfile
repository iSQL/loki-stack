# Bake config into the image. Coolify's build container can't share file
# paths with the host docker daemon, so bind mounts to `./loki/config.yml`
# and Compose `configs: file: ./...` both resolve to paths the host can't
# see. COPY at build time sidesteps both problems and keeps the repo as
# the single source of truth.
FROM grafana/loki:3.3.2
COPY config.yml /etc/loki/config.yml
