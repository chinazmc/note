services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - 9090:9090
    restart: unless-stopped
    privileged: true
    environment:
      - https_proxy=
      - http_proxy=
    volumes:
      - ./prometheus:/etc/prometheus
      - prom_data:/prometheus
    network_mode: "slirp4netns:allow_host_loopback=true"
  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - 3000:3000
    restart: unless-stopped
    privileged: true
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=grafana
      - https_proxy=
      - http_proxy=
    volumes:
      - ./grafana:/etc/grafana/provisioning/datasources
      - grafana_data:/var/lib/grafana
    network_mode: "slirp4netns:allow_host_loopback=true"
volumes:
  prom_data:
  grafana_data: