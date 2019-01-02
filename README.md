# Home Assistant docker-compose

This is my setup for home assistant using docker-compose. This is tested and runs fine on any Ubuntu distribution from 16.04 onwards.

It contains the following components:
- [Home Assistant](https://hub.docker.com/r/homeassistant/home-assistant/) (duh)
- [Postgresql](https://hub.docker.com/_/postgres). The database Home Assistant should use
- [InfluxDB](https://hub.docker.com/_/influxdb). The database Home Assistant writes sensory information to such that Grafana can read it
- [Grafana](https://hub.docker.com/r/grafana/grafana). Dashboarding
- [Mosquitto](https://hub.docker.com/_/eclipse-mosquitto). MQTT server
- [LetsEncrypt](https://hub.docker.com/r/linuxserver/letsencrypt). Service to setup and maintain SSL for Home Assistant
- [Node Red](https://hub.docker.com/r/nodered/node-red-docker): Alternative for the yaml automations in Home Assistant

A brief setup, requirements and list of quirks is compiled below per service. Most of these are a culmination of many forum posts of which is was unable to find all resources.
Might you find the root post, please let me know and I will attribute it :).

## Home Assistant

Doesn't require much setup. The home directory will be `/home/<user>/.homeassistant`. Change `<user>` to your username.
To leverage all components you must have at least the following in your `configuration.yaml`.

```yaml
http:
  api_password: !secret http_password
  base_url: !secret base_url
api:

influxdb:
  host: localhost
  port: 8086
  database: homeassistant
  username: !secret influxdb_username
  password: !secret influxdb_password
  
recorder:
  db_url: !secret postgres_url 
```

The `postgres_url` should be in this format: 
```bash
postgresql://username:password@host/db
```
For example
```bash
postgresql://homeassistant:mypass@localhost/home-assistant
```

## Postgresql
Should run without issues. Must be in `host` network mode for Home Assitant to be able to connect. You could setup a docker network, but this setup exposes Home Assistant to the outside world.
The persistent directory will be `/home/<user>/.postgres`. Change `<user>` to your username.

## InfluxDB
Should run without issues. Must be in `host` network mode for Home Assitant to be able to connect
The persistent directory will be `/home/<user>/.influxdb`. Change `<user>` to your username.

## Grafana
The persistent directory will be `/home/<user>/.grafana`. Change `<user>` to your username.
The grafana docker container runs on user:group `472:472`. The persistent volume must therefor be `chmod` as that user. Use `sudo chown -R 472:472 /home/<user>/.grafana` to achieve that.

## LetsEncrypt
This is the most annoying one. It doesn't work out the box as described on their website
The persistent directory will be `/home/<user>/.letsencrypt`. Change `<user>` to your username.

Also in the `docker-compose.yml` change `<email>` and `<url>` to your email address and your DNS name. Register one for free at [https://www.duckdns.org/](https://www.duckdns.org/)

After starting the LetsEncrypt once you have to modify `/home/<user>/.letsencrypt/config/nginx/site-confs` to add this:

```yaml
### HOMEASSISTANT ##############################################################
server {
        listen 443 ssl;

        root /config/www;
        index index.html index.htm index.php;

        server_name <your-dns>;

        include /config/nginx/ssl.conf;

        client_max_body_size 0;

        location / {
                proxy_set_header Host $host;
                proxy_redirect http:// https://;
                proxy_http_version 1.1;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_buffering               off;
                proxy_ssl_verify              off;
                proxy_pass http://<host-ip>:8123;
        }
}
```

Without this nginx won't forward the DNS subdomain to Home Assistant. Replace `<your-dns>` and `<host-ip>` with your values. For every subdomain (perhaps you want to expose mqtt!?) you can add above configuration for each subdomain and add it to the SUBDOMAINS list, i.e.:
```yaml
- SUBDOMAINS=hass,mqtt,other
```

Restart the container and if everything is right home assistant should be available on your own DNS address.

## Node Red
Runs as user `1000` in the docker container. For some docker might run as `root`. The docker user for Node Red has therefor been configured as `root` such that Node Red has rights to read/write the persistent storage.
The persistent directory will be `/home/<user>/.nodered`. Change `<user>` to your username.

After running the container, step into the container with `docker exec -it nodered bash` and run
```bash
cd /data
npm install node-red-contrib-home-assistant
```

exit the container and restart the container.
