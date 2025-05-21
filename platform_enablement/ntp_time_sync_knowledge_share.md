NTP (Network Time Protocol) is for keeping hosts in sync with the world clock. Time synchronization is important for time sensitive operations, such as log keeping and time stamps, and is highly recommended.

The OpenShift Container Platform installation playbooks install, enable, and configure the ntp package to provide NTP service by default.

You can verify that a host is configured to use NTP by running `timedatectl`:
```
$ timedatectl
      Local time: Thu 2017-12-21 14:58:34 UTC
  Universal time: Thu 2017-12-21 14:58:34 UTC
        RTC time: Thu 2017-12-21 14:58:34
       Time zone: Etc/UTC (UTC, +0000)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a
```
If both `NTP enabled` and `NTP synchronized` are `yes`, then NTP synchronization is active.

To install the `ntp` package, run the following command:
```
# timedatectl set-ntp true
```

To install the `chrony` package, run the following commands:
```
# yum install chrony
# systemctl enable chronyd --now
```

## Relevant Documentation
https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/system_administrators_guide/chap-configuring_the_date_and_time#sect-Configuring_the_Date_and_Time-timedatectl-NTP
