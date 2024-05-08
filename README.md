# Learn Promtail

## Install

```
wget https://github.com/grafana/loki/releases/download/v2.9.8/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
cd promtail-linux-amd64

sudo chmod a+x "promtail-linux-amd64"

cp promtail-linux-amd64 /usr/local/bin/
```

```
mkdir /etc/promtail
```


vi /etc/promtail/config-promtail.yml
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: 'http://localhost:3100/loki/api/v1/push'

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log
```
This default configuration was copied from https://raw.githubusercontent.com/grafana/loki/master/clients/cmd/promtail/promtail-local-config.yaml.  
There may be changes to this config depending on any future updates to Loki.



## Configure Promtail as a Service

```
sudo useradd --system promtail
```


vi /etc/systemd/system/promtail.service
```
[Unit]
Description=Promtail service
After=network.target

[Service]
Type=simple
User=promtail
ExecStart=/usr/local/bin/promtail-linux-amd64 -config.file /etc/promtail/config-promtail.yml

[Install]
WantedBy=multi-user.target
```


```
systemctl daemon-reload

systemctl start promtail
systemctl enable promtail
systemctl status promtail
```

Now, since this example uses Promtail to read system log files, the promtail user won't yet have permissions to read them.
```
usermod -a -G adm promtail
```

Verify that the user is now in the adm group
```
id promtail

systemctl restart promtail
```


## Configure Firewall

When your Promtail server is running, it will be accessible remotely. If you only want localhost to be able to connect, then type
```
iptables -A INPUT -p tcp -s localhost --dport 9080 -j ACCEPT
iptables -A INPUT -p tcp --dport 9080 -j DROP
iptables -L
```

Also, I have hard coded my Promtail service to use port 9097 for gRPC communications. This port may also be accessible across the internet. To close it using iptables, then use,
```
iptables -A INPUT -p tcp -s <your servers domain name or ip address> --dport 9097 -j ACCEPT
iptables -A INPUT -p tcp -s localhost --dport 9097 -j ACCEPT
iptables -A INPUT -p tcp --dport 9097 -j DROP
iptables -L
```

Be sure to back up your iptables rules if you installed iptables-persistent in the last section.
```
iptables-save > /etc/iptables/rules.v4
iptables-save > /etc/iptables/rules.v6
```

Check
```
curl "127.0.0.1:9080/metrics"
```



## Troubleshooting

msg="error creating promtail" error="open /tmp/positions.yaml: permission denied"
```
chown promtail:promtail /tmp/positions.yaml
```

OR

```
getfacl /var/log/fail2ban.log

This is typically he result of adding a log after running setfacl above. Running setfacl again gives the promtail-user read access to the log:
sudo setfacl -R -m u:promtail:rX /var/log
```




If you set up Promtail service with to run as a specific user, and you are using Promtail to view adm and you don't see any data in Grafana, but you can see the job name, then possibly you need to add the user Promtail to the adm group.
```
usermod -a -G adm promtail
```
You may need to restart the Promtail service and checking again its status


Error: Account not available
If you try to run promtail with runuser
```
sudo runuser -l promtail -c "promtail -log.level=debug -config.file=/etc/loki/promtail.yaml"
and you get: This account is currently not available.

there is no valid shell for the promtail user. Add a shell with the chsh-command:
$ sudo chsh -s /bin/bash promtail
```





If you see the error "found character that cannot start any token" than that is likely to mean you have a tab somewhere in the YML indenting one of the tokens. Replace it with spaces.

On Linux, you can check the syslog for any Promtail related entries by using the command,
```
tail -f /var/log/syslog | grep promtail
```

##  Pipe data to Promtail

```
send logs to a local Loki instance:
cat my.log | promtail --stdin  --client.url http://127.0.0.1:3100/loki/api/v1/push

You can also add additional labels from command line using:
cat my.log | promtail --stdin  --client.url http://127.0.0.1:3100/loki/api/v1/push --client.external-labels=k1=v1,k2=v2

This will add labels k1 and k2 with respective values v1 and v2.

To force Promtail to re-send log messages, delete the positions file (default location /tmp/positions.yaml).

```

## Test Loki

First, start Promtail
```
sudo runuser -l promtail -c "/usr/local/bin/./promtail-linux-amd64 -log.level=debug -config.file=/etc/loki/promtail.yaml"
```

Note: If you try to run promtail with runuser and you get This account is currently not available. there is no valid shell for the promtail user. Add a shell with the chsh-command:
```
sudo chsh -s /bin/bash promtail
```

Make sure there are actual events in the logs. See if anything appears in Loki. If not, after at least 60 seconds, start (or restart) Loki. From the command line:
```
sudo loki -log.level=debug -config.file=/etc/loki/config.yml
```

Next, see if Promtail is working
```
http://server-ip-address:9080/targets
http://server-ip-address:9080/service-discovery
```


