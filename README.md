docker-compose files and configs for my monitoring stack. Alloy collects metrics/logs, Loki processes logs, Prometheus processes metrics, and Grafana makes it all pretty.

Greatly referenced from the wonderful https://blog.fixermark.com/posts/2025/monitoring-home-network-grafana-loki-prometheus-alloy/ which is like the only no-nonsense, actually-written-by-a-human guide I found on doing this.

notes:
- Generally speakng, metrics are pull, logs are push. However...
    - Logs being "push" means Alloy is required on all nodes.
    - Because we're running Alloy anyway, we may as well collect our metrics with it, making our metrics "push" instead of the usual "pull". For something like Iocaine, you'd probably be running a local Prometheus in agent mode anyways, and Alloy can replace that.
        - I'll mention [Prometheus federation](https://prometheus.io/docs/prometheus/latest/federation/) here, but it's really not necessary if you're running Alloy. If you're _not_ running Alloy, then hierarchical federation is exactly what you want.
- docker-compose.yml runs only on host
- docker-compose-alloy.yml runs on host and nodes
- I have persistent storage for all of these -- Alloy doesn't necessarily need it, but I want it to continue collecting even if Loki goes down. They have driver_opts defined on each of the volumes, change as desired (I wanted to use named volumes but also store them on an attached Hetzner volume for the host)

caveats:
- I use HestiaCP for several of my servers. You will need to install Alloy/Prometheus with apt and run it as a systemd service in that kind of scenario, but your configs are gonna look pretty much the same.

host machine setup:
1. `cp -r example_configs config`
2. `cp .env_example .env`
3. Update .env file with volume paths (mine are i.e. `/mnt/storage/grafana`) and the hostname of the server
2. Update grafana configs:
    1. `config/grafana/provisioning` is where various things that you want at runtime will live. These can't be edited from within Grafana itself, so keep that in mind.
        - `datasources/ds.yml` does not need to be changed unless you are adding sources outside of local Prometheus and Loki
        - `dashboards/` put your dashboards you want on runtime here
        - `plugins/` put your plugins you want on runtime here
        - `alerting/` put your alert notifications you want on runtime here, [see documentation](https://grafana.com/docs/grafana/latest/alerting/set-up/provision-alerting-resources/file-provisioning/)
        - If you update these folders, you can use [admin API routes](https://grafana.com/docs/grafana/latest/developer-resources/api-reference/http-api/api-legacy/admin/#reload-provisioning-configurations) to load the changed data without recreating the container etc
    2. I use secrets for the admin email/user/password (`/secrets/gf_admin_email.txt` etc). Change it to .env variables if you want, I'm not your mom.
    3. Grafana has a million different config options you could add to the `docker-composer.yml`. I set it to some fairly sensible ~~if somewhat paranoid~~ defaults, see [here](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/) for the rest of them.
        - If you have a _lot_ of configs, you may to consider binding an ini file with `/config/grafana/grafana.ini:/etc/grafana/grafana.ini`
3. Update Loki configs:
    1. I have Loki in monolithic mode with persistent storage and the compactor enabled. It deletes logs after 7 days. Change in the `config/loki/loki-config.yml` if desired.
    2. There's a lot of extra stuff I have in the loki-config.yml that you probably don't need if you're not dealing with a giant firehose like me. Feel free to review and pare down if needed.
    3. If Loki is getting OOM killed, adjust/increase values in `limits_config`.
3. Update Alloy configs:
    1. `cd config/alloy`
    2. If you're on Ubuntu, `cat config-ubuntu.alloy >> config.alloy` to capture systemd
    4. To add Prometheus, `cat config-prometheus.alloy >> config.alloy`
    5. Update `prometheus.remote_write.endpoint.host` and `loki.write.endpoint.url` to your actual domains as needed
4. Update Prometheus configs:
    - Realistically, this shouldn't need updating (e.g. no need for `scrape_configs`), because Alloy is doing our scraping.
    - However, if you're having performance issues, you may want to adjust the global `evaluation_interval`, `body_size_limit`, etc. 

child node setup:
You can `git checkout child-node` to have just the relevant files if you want.
1. Remove `alloy_data` if persistent data is not desired (or just `driver:` and `driver_opts:` if you do not care where it goes)
2. `cp .env_example .env`
3. Update .env file with hostname of server
4. `cd config/alloy`
2. If you're on Ubuntu, `cat config-ubuntu.alloy >> config.alloy` to capture systemd
7. To add Prometheus, `cat config-prometheus.alloy >> config.alloy`
8. Update `prometheus.remote_write.endpoint.host` and `loki.write.endpoint.url` to your actual domains as needed