---
title: Authmap
featured: true
meta: A service that reads attempted SSH authentications and logs them to influxdb.
icon: mdi:earth
layout: page
---

{:refdef: style="text-align: center;"}
![Mapped SSH Requests](/assets/img/authmap.png)
{: refdef}

Shortly after opening up my home server's ssh port to the world (with only public/private key auth enabled), I started getting a flood of attempted ssh requests from all over the place. Out of curiosity, I wanted to see them plotted out on my Grafana dashboard so I started putting together a small service that reads SSH authentication attempts, parses them, and pushes them into influxdb.

This project originally started out as my first foray into Rust. Pretty soon after that, I began experimenting with Golang for systems programming and decided to port what I had made so far into Go. Links for both versions below:
- [Authmap (Rust)](https://github.com/tgiv014/authmap)
- [Authmap (Go)](https://github.com/tgiv014/authmap_go)

# Goals
I have a few goals to refine this project into something more useful:
- ~~Remove every instance of hardcoding and read all configurables from a YAML file.~~
  - Done! Now, the only hardcoded configuration item is the config's location. The application will also generate a default configuration automatically if a config is not found.
- ~~Allow grouping requests from the same IP/location in a smart way.~~
  - Done! The `ip` field is now a tag. This allows grouping by ip and aggregating results within those groups.
- Automate downloading the MaxMind GeoIP database.
  - This will require user interaction since the database has licensing requirements now.
- Make the docker container for this package more accessible.