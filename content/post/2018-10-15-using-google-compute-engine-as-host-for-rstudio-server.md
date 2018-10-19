---
title: Using Google Compute Engine as host for RStudio Server
author: Seth Russell
date: '2018-10-15'
slug: using-google-compute-engine-as-host-for-rstudio-server
categories:
  - R
tags:
  - blog
  - R
  - google cloud
description: ''
---

*Note:* Last updated October 19, 2018. This post is a work in progress.

[RStudio Server](https://www.rstudio.com/products/rstudio-server-pro/)  ([download](https://www.rstudio.com/products/rstudio/download-server/)), is feature comparable to the desktop software [RStudio IDE](https://www.rstudio.com/products/rstudio/). RStudio Server is available under two different licensing models: "Open Source Edition: (AGPL v3), and "Commercial License."

Google Cloud has a wide range of services available. Everything from machine learning APIs to development environments to database as a service. However, one thing they don't have is a good R environment for development of R packages or analysis. I've found the most flexible option to get a great R environment on Google Cloud is to use a Google Compute Engine instance (GCE) - Google's cloud hosted virtual machine platform.

In this post, I'll be setting up the Open Source Edition of RStudio Server. Total estimated cost per month if the system is running 24x7: $202.68/month. Cost is much less if you only run the system as needed (business hours only would cost about $45/month)


Steps:

1. Create new GCE
  1. Navigate to https://console.cloud.google.com. If you don't already have an account, you can setup a free account with $300 credits for 365 days.
  1. From the navigation menu (top left), select "Compute Engine" (under 'COMPUTE') -> "VM Instances"
  1. Click "CREATE INSTANCE" at the top of the page
  1. Name your instance as desired.
  1. Pcik desired zone. I selected us-central1 (as that zonehas GPU availability should I want that feature in the future)
  1. **Machine type**: I selected **8 CPU 30GB RAM (n1-standard-8)** based on estimated resources needed. This host will support 6 Data Scientist; you can always scale this up or down as needed after VM creation.
  1. **Boot Disk**: Click change - select **Ubuntu 18.04 LTS minimal** - LTS minimal installs very few packages by default, keeping disk usage minimized. Most Linux distributions are free to use, but not all OSes are; some have an hourly charge.
  1. **Boot Disk**: Recommend setting size to 50GB and selecting **Boot Disk Type** to SSD. While an SSD will cost $8/more per month than standard persistent disk when running 24x7, it also performs much better when using local files.
  1. **Firewall**: Check both boxes to allow HTTP and HTTPS traffic.
  1. Click "Create"
  1. **Static IP Address setup**:
      1. Once machine is up (takes perhaps 30 seconds), click the machine name
      1. Scroll down to the "Network Interfaces" section and click "View Details" in the "Network Details" column.
      1. From the left menu "VPC Network", select "External IP addresses"
      1. In the "Type" column, click the drop down to change from "Ephemeral" to "Static". A static ip reservation results in a constant but small ongoing charge.
1. Basic machine setup: One the VM Instance page, in the "Connect" column, click "SSH". Once the SSH window pops up, run these commands
  1. `sudo apt-get install htop tmux dialog vim lsof less` This installs a few basic tools that I find to be helpful
  1. `sudo adduser <name>` Add users as appropriate. RStudio Server uses local accounts to control access to the web UI.
1. Install R and RStudio. I selected to install R 3.5.x by following instructions at https://cran.rstudio.com/bin/linux/ubuntu/README.html
  1. `sudo apt-get update`
  1. `sudo apt-get upgrade`
  1. `apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E298A3A825C0D65DFD57CBB651716619E084DAB9`
  1. add this line to /etc/apt/sources.list: `deb https://cloud.r-project.org/bin/linux/ubuntu bionic-cran35/`
  1. `sudo apt-get install r-base r-base-dev gdebi-core`
  1. `wget https://download2.rstudio.org/rstudio-server-1.1.456-amd64.deb`
  1. `sudo gdebi rstudio-server-1.1.456-amd64.deb`
1. Run R and install additional desired packages. *Note:* Installing packages as root installs those packages for all uesrs.
  1. `sudo apt-get install libssl-dev libcurl4-openssl-dev libxml2-dev wget` This fullfills some dependency requirements for desired packages.
  1. `sudo R`
  1. `install.packages(c('tidyverse', 'bigrquery', 'devtools'))`
  1. Verify packages have been installed correctly: `require("dplyr")` and `require('bigrquery')` etc.
1. Install Nginx and add Let's Encrypt SSL cert https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx 
  1. `sudo apt-get install nginx software-properties-common`
  1. `sudo add-apt-repository ppa:certbot/certbot`
  1. `sudo apt-get update`
  1. `sudo apt-get install python-certbot-nginx`
  1. Create a new nginx configuration file by copying the default configuration:
      1. `sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/rstudio-server`
      1. Remove the default site from being served, but keep the configuration for reference: `sudo rm /etc/nginx/sites-enabled/default`
      1. Make sure Nginx still works - `sudo systemctl restart nginx`
  1. `sudo certbot --nginx`
1. Setup Nginx to proxy RStudio Server via https port. Follow instructions similar to https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-with-ssl-as-a-reverse-proxy-for-jenkins and https://support.rstudio.com/hc/en-us/articles/200552326-Configuring-the-Server/
  1. Modify the :443 portion to do the https proxy - this goes in a location{} section; the key lines are:
      ```
      proxy_set_header        Host $host;
      proxy_set_header        X-Real-IP $remote_addr;
      proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto $scheme;
      # adjust these two lines to match your RStudio Server port
      proxy_pass          http://localhost:8787;
      proxy_redirect      http://localhost:8787 $scheme://$http_host/;
      ```
      
  1. In order to get the websockets component working, you need to modify /etc/nginx/nginx.conf and add the following section inside the http{} block:
      ```
      map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
      }
      ```
      
  1. Add the following lines to your server{} block:
      ```
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
      proxy_read_timeout 7d; # want something long so websocket doesn't automatcailly close on user
      ```
      
  1. Symlink your `/etc/nginx/sites-available` file to `/etc/nginx/sites-enabled` sites-enabled is the directory that the nginx server will look in when starting up.
  1. Resulting nginx configuration file should look like: https://gist.github.com/magic-lantern/1b5e11c3cf5964b69e8e7824df015c5d
  1. Before restarting nginx, test your configuration by running `nginx -t`. Fix any errors before restarting with `systemctl restart nginx`
  

Some URLs for additional information:

* General setup: https://towardsdatascience.com/running-jupyter-notebook-in-google-cloud-platform-in-15-min-61e16da34d52
* Static IP Address: https://cloud.google.com/compute/pricing?hl=en_US&_ga=2.8427253.-1719922974.1503442674#ipaddress
* Network Tier Pricing: https://cloud.google.com/network-tiers/pricing
* Nginx as a reverse proxy: https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-with-ssl-as-a-reverse-proxy-for-jenkins
* Starting and Stoping the RStudio Server Service: https://support.rstudio.com/hc/en-us/articles/200532327-Managing-the-Server
* RStudio Server setup using PAM and Apache : https://jstaf.github.io/2018/06/20/rstudio-server-semi-pro.html
