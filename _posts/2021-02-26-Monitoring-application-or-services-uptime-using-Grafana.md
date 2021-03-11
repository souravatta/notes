---
layout: post
topic: grafana
permalink: /:categories/:title
---
We are going to monitor application or service uptime or downtime using Grafana. This can be achieved by keeping track of the http_status_code, **uptime** means http_status_code of 200 and **downtime** means any http_status_code other than 200.

Tools included are:

- Grafana
- Prometheus
- BlackBox Exporter

Pre-requisite:

- Ubuntu Server (LTS 18.04 or above)
- Grafana should be installed (Please refer this post to install Grafana)

---

Lets quickly dive into installing and setting up Prometheus and BlackBox Exporter.

<h3><u>Installing Prometheus</u></h3>

1. Update the package list with the following command
```bash
sudo apt-get update
sudo apt-get upgrade
```
2. Install Prometheus
```bash
sudo apt-get install prometheus
```
3. Once the installation is complete, start Prometheus by running the below commands:
```bash
sudo systemctl start prometheus
sudo systemctl enable prometheus
```
4. Check the status of the Prometheus service
```bash
sudo systemctl status prometheus
```
Output:
```
● prometheus.service - Monitoring system and time series database
     Loaded: loaded (/lib/systemd/system/prometheus.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2021-02-26 08:17:58 UTC; 3h 16min ago
       Docs: https://prometheus.io/docs/introduction/overview/
   Main PID: 24408 (prometheus)
      Tasks: 9 (limit: 1160)
     Memory: 84.2M
     CGroup: /system.slice/prometheus.service
             └─24408 /usr/bin/prometheus
```
5. Try to access the prometheus at address: http://localhost:9090/
>Note: If you have installed Prometheus on AWS, modify the security group to allow the port 9090 to be accessible from outside.

<img src="assets/images/prometheus_dashboard.png" alt="Prometheus Dashboard">

---

<h3><u>Installing BlackBox Exporter</u></h3>

1. Update the package list with the following command
```bash
sudo apt-get update
```
2. Download the BlackBox Exporter
```bash
cd ~
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.18.0/blackbox_exporter-0.18.0.linux-amd64.tar.gz
```
3. Extract the binaries
```bash
tar -xvf blackbox_exporter-0.14.0.linux-amd64.tar.gz
```
4. Create a Blackbox Exporter user and copy the extracted binaries to the new user's bin directory. Also, change directory ownership.
```bash
sudo useradd --no-create-home --shell /bin/false blackbox_exporter
sudo mv blackbox_exporter-0.14.0.linux-amd64/blackbox_exporter /usr/local/bin/blackbox_exporter
sudo chown blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter
```
5. Create a configuration file for Blackbox Exporter.
```bash
sudo mkdir /etc/blackbox_exporter
sudo nano /etc/blackbox_exporter/blackbox.yml
```
Copy the below configuration into `blackbox.yml`:
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: []
      method: GET
```
6. To start the Blackbox Exporter, we need to create a systemd service.
```bash
sudo nano /etc/systemd/system/blackbox_exporter.service
```

    Copy the below lines inside `blackbox_exporter.service`:
    ```bash
      [Unit]
      Description=Blackbox Exporter
      Wants=network-online.target
      After=network-online.target

      [Service]
      User=blackbox_exporter
      Group=blackbox_exporter
      Type=simple
      ExecStart=/usr/local/bin/blackbox_exporter --config.file /etc/blackbox_exporter/blackbox.yml

      [Install]
      WantedBy=multi-user.target
      ```
7. Update Blackbox Exporter's configuration file permissions.
```bash
chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter/blackbox.yml
```
8. Start the Blackbox Exporter.
```bash
sudo systemctl daemon-reload
sudo systemctl start blackbox_exporter
sudo systemctl enable blackbox_exporter
```
9. Check the status of BlackBox Exporter
```bash
systemctl status blackbox_exporter
```

    Output:
```
● blackbox_exporter.service - Blackbox Exporter
     Loaded: loaded (/etc/systemd/system/blackbox_exporter.service; disabled; vendor preset: enabled)
     Active: active (running) since Fri 2021-02-26 06:16:23 UTC; 6h ago
   Main PID: 3599 (blackbox_export)
      Tasks: 8 (limit: 1160)
     Memory: 12.9M
     CGroup: /system.slice/blackbox_exporter.service
             └─3599 /usr/local/bin/blackbox_exporter --config.file /etc/blackbox_exporter/blackbox.yml
```

---

<h3>Configuring Prometheus</h3>

We will now configure the prometheus to scrape data from blackbox_exporter, to monitor our flask app running on port 5000.

Open the file `prometheus.yml` using your favorite editor, eg nano
```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add the below lines in the file:

```yaml
- job_name: 'blackbox'
   metrics_path: /probe
   params:
     module: [http_2xx]
   static_configs:
     - targets:
       - http://localhost:5000
   relabel_configs:
     - source_labels: [__address__]
       target_label: __param_target
     - source_labels: [__param_target]
       target_label: instance
     - target_label: __address__
       replacement: localhost:9115
```

Restart the prometheus service, with the following command:
```bash
sudo systemctl restart prometheus
```
---
<h3>Configuring Grafana and Creating Dashboard</h3>

Assuming Grafana is already installed, if not, please follow the steps to install Grafana here in this post.

<h5>Configure the Data Source:</h5>

1. Go to `Settings` >> `Data Source`. Click on `Add data source`
2. Fill in the following details:
    - Name: Prometheus Data Source
    - Type: Prometheus
    - URL: http://localhost:9090
3. Click on `Save & Test`. You can see the msg `Data source is working`
4. Click on `Back`

 <img src="assets/images/Prometheus_Data_Source.png" alt="Prometheus Data Source">

 <h5>Creating Dashboard</h5>

 1. Click on `**+**` icon and the `Dashboard`
 2. Click on `Add panel` and select the `Graph`
 3. Click on `Panel Title` >> `Edit`
 4. Select the Data Source: `Prometheus Data Source` and Query: `probe_http_status_code{instance="http://localhost:5000"}`, Legend Format: `HTTP Code: 200`

 <h5>Alerting using Slack</h5>

 1. **<u>Configuring Slack for Alert Notification</u>**

    * Create a new workspace in Slack.
    * Under Apps, click on **Add apps**
    * Click on **App directory** >> **Build** >> **Create a custom app**
    * Give a name to the App and select the workspace. Click on **Create App**
    * Once the App is created, under **Features**, click on **Incoming Webhooks**
    * Under **Webhook URLs for Your Workspace**, click on **Add New Webhook to Workspace**
    * Select the channel, where the App will post the notifications. Then, click on **Allow**
    * Copy the Webhook Url for the particular channel

    >**Note:** Don't forget to "Toggle On" the Activate Incoming Webhooks

 2. **<u>Configuring Grafana for Alert</u>**

    * 
