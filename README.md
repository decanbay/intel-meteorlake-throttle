# intel-meteorlake-throttle
Alternative simple approach to intel pstate to cap cpu frequencies safely for Core Ultra 7 Meteor lake CPUs. 

My thinkpad was running very hot in the balanced power mode which does not cap the frequencies, powersave mode aggressively caps the frequencies and hurts the performance.

The below is to create a hybrid approach that does not hurt the performance much and runs -40 degrees cooler compared to balamced power mode on my thinkpad with Core Ultra 7 155h CPU.

To disable intel_pstate
```bash
sudo nano /etc/default/grub
```
change the line 
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```
to
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_pstate=passive"
```
then
```bash
sudo update-initramfs -u
```
and
```bash
sudo update-grub
```
Now that the pstate is disabled, a reboot is required

```bash
sudo reboot
```
To see your core configuration and their max design frequencies 
```bash
lscpu -e=CPU,CORE,SOCKET,NODE,ONLINE,MAXMHZ | sort -k6 -n
```
For example my machine returns:

| Max MHz | Core Type | CPUs |
| ------- | -------------- | ---------- |
| 4800    | P-core (big)     | 1, 2, 3, 4 |
| 4500    | P-core (mid)     | 0, 5–11    |
| 3800    | E-core (fast)    | 12–19      |
| 2500    | E-core (slow)    | 20–21      |

since  155h CPu has 6 P-cores total (2x@4.8 GHz + 4x@4.5 GHz) and 8 E-cores (6x@3.8 GHz + 2x@2.5 GHz)

I have aimed for the following max freqs

| Max Cap | Core Type        | Count |
| ------- | ---------------- | ----- |
| 3.5 GHz | P-cores @4.8 GHz | 2     |
| 3.3 GHz | P-cores @4.5 GHz | 6     |
| 3.0 GHz | E-cores @3.8 GHz | 8     |
| 2.5 GHz | E-cores @2.5 GHz | 2     |

You can directly run this script which will not persist after reboot
```bash
declare -A cpu_caps=(
  [4800000]=3500   # cap big P-cores to 3.5 GHz
  [4500000]=3300   # cap mid P-cores to 3.3 GHz
  [3800000]=3000   # cap fast E-cores to 3.0 GHz
  [2500000]=2500   # cap slow E-cores to 2.5 GHz
)

for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
    core=$(basename "$cpu" | sed 's/cpu//')
    max=$(cat "$cpu/cpufreq/cpuinfo_max_freq")  # in kHz

    mhz_cap=${cpu_caps[$max]}
    if [ -n "$mhz_cap" ]; then
        sudo cpufreq-set -c "$core" -u "${mhz_cap}MHz"
    else
        echo "No cap set for cpu$core (max=$((max/1000)) MHz)"
    fi
done
```

To make it persist after reboot create a systemd service:
```bash
sudo nano /etc/systemd/system/cpu-cap.service
```
paste this inside
```bash
[Unit]
Description=Set CPU frequency caps per core
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/set-cpu-caps.sh

[Install]
WantedBy=multi-user.target
```

```bash
sudo nano /usr/local/bin/set-cpu-caps.sh
```
paste the previous script inside, ie
```bash
#!/bin/bash
declare -A cpu_caps=(
  [4800000]=3500   # cap big P-cores to 3.8 GHz
  [4500000]=3300   # cap mid P-cores to 3.5 GHz
  [3800000]=3000   # cap fast E-cores to 2.8 GHz
  [2500000]=2500   # cap slow E-cores to 2.2 GHz
)

for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
    core=$(basename "$cpu" | sed 's/cpu//')
    max=$(cat "$cpu/cpufreq/cpuinfo_max_freq")  # in kHz

    mhz_cap=${cpu_caps[$max]}
    if [ -n "$mhz_cap" ]; then
        sudo cpufreq-set -c "$core" -u "${mhz_cap}MHz"
    else
        echo "No cap set for cpu$core (max=$((max/1000)) MHz)"
    fi
done
```
then make it executable
```bash
sudo chmod +x /usr/local/bin/set-cpu-caps.sh
```
and finally enable the service
```
sudo systemctl daemon-reexec
sudo systemctl enable cpu-cap.service
```

To verify the settings
```bash
cpufreq-info | grep "current policy"
```
This for example returns 
```bash
  current policy: frequency should be within 400 MHz and 3.50 GHz.
  current policy: frequency should be within 400 MHz and 3.80 GHz.
  current policy: frequency should be within 400 MHz and 3.80 GHz.
  current policy: frequency should be within 400 MHz and 3.80 GHz.
  current policy: frequency should be within 400 MHz and 3.80 GHz.
  current policy: frequency should be within 400 MHz and 3.50 GHz.
  current policy: frequency should be within 400 MHz and 3.50 GHz.
  current policy: frequency should be within 400 MHz and 3.50 GHz.
  current policy: frequency should be within 400 MHz and 3.50 GHz.
  current policy: frequency should be within 400 MHz and 3.50 GHz.
  current policy: frequency should be within 400 MHz and 3.50 GHz.
  current policy: frequency should be within 400 MHz and 3.50 GHz.
  current policy: frequency should be within 400 MHz and 2.80 GHz.
  current policy: frequency should be within 400 MHz and 2.80 GHz.
  current policy: frequency should be within 400 MHz and 2.80 GHz.
  current policy: frequency should be within 400 MHz and 2.80 GHz.
  current policy: frequency should be within 400 MHz and 2.80 GHz.
  current policy: frequency should be within 400 MHz and 2.80 GHz.
  current policy: frequency should be within 400 MHz and 2.80 GHz.
  current policy: frequency should be within 400 MHz and 2.80 GHz.
  current policy: frequency should be within 400 MHz and 2.20 GHz.
  current policy: frequency should be within 400 MHz and 2.20 GHz.
```
which confirms that the script is effective. 

---

To revert back simply enable the pstate back and disable the service

change the line 
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_pstate=passive"
```
to
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```
then
```bash
sudo update-initramfs -u
```
and
```bash
sudo update-grub
```
```bash
sudo systemctl disable cpu-cap.service
```
and reboot
```bash
sudo reboot
```
finally check the frequencies
```bash
cpufreq-info | grep "current policy"
```

