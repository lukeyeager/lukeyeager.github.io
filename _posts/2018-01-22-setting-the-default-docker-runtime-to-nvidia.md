---
layout: post
title: Setting the default docker runtime to nvidia
---

With nvidia-docker2, it is now possible to change your default docker runtime to nvidia, instead of runc.
This means you can use any docker tool (e.g. docker-compose) and get GPUs visible inside your containers by default.

> [nvidia-docker](https://github.com/NVIDIA/nvidia-docker) v2 depends on [libnvidia-container](https://github.com/NVIDIA/libnvidia-container) which is still just an alpha release.
> So, use these instructions at your own risk, and think carefully about the security implications before using this in production.

## Installation

Follow [these instructions](https://github.com/NVIDIA/nvidia-docker/wiki/Installation-(version-2.0)).

For arch linux, you can use [this AUR package](https://aur.archlinux.org/packages/nvidia-docker2/).

## Setting the default runtime

{% highlight bash %}
# Update the default configuration and restart
pushd $(mktemp -d)
(sudo cat /etc/docker/daemon.json 2>/dev/null || echo '{}') | \
    jq '. + {"default-runtime": "nvidia"}' | \
    tee tmp.json
sudo mv tmp.json /etc/docker/daemon.json
popd
sudo systemctl restart docker

# No need for nvidia-docker or --engine=nvidia
docker run --rm -it nvidia/cuda nvidia-smi
{% endhighlight %}
