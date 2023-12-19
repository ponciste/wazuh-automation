# Wazuh Web Shell Detection and Prevention Configuration

## Changes to auditd

**PATH**: `/etc/audit/rules.d/`

**FILE**: `audit.rules`

### Add auditd rule to detect files uploaded to `/var/www/html/uploads`
```console
  -w /var/www/html/uploads -p w -k audit-wazuh-w
```
### Auditd rules that detect command execution from user www-data
```console
  -a always,exit -F arch=b32 -S execve -F uid=33 -F key=webshell_command_exec
  -a always,exit -F arch=b64 -S execve -F uid=33 -F key=webshell_command_exec
```
### Auditd rules that detect network connections from user www-data
```console
  -a always,exit -F arch=b64 -S socket -F a0=10 -F euid=33 -k webshell_net_connect
  -a always,exit -F arch=b64 -S socket -F a0=2 -F euid=33 -k webshell_net_connect
  -a always,exit -F arch=b32 -S socket -F a0=10 -F euid=33 -k webshell_net_connect
  -a always,exit -F arch=b32 -S socket -F a0=2 -F euid=33 -k webshell_net_connect
```
### Update Configuration

```console
  systemctl restart auditd
```

## Changes to `ossec.conf`

**PATH**: `/var/ossec/etc`

**FILE**: `ossec.conf`

### Add Audit Logs
```xml
  <!--  Add Audit Logs -->
  <localfile>
    <log_format>audit</log_format>
    <location>/var/log/audit/audit.log</location>
  </localfile>
```

### Detect Webshell Connection
```xml
  <!-- Detect webshell connections by checking open TCP connections -->
  <localfile>
    <log_format>full_command</log_format>
    <command>ss -nputw | egrep '"sh"|"bash"|"csh"|"ksh"|"zsh"' | awk '{ print $5 "|" $6 }'</command>
    <alias>webshell connections</alias>
    <frequency>30</frequency>
  </localfile>
```

### Active Response
```xml
  <!-- Add custom kill-webshell script command -->
  <command>
    <name>kill-webshell</name>
    <executable>kill-webshell.sh</executable>
    <timeout_allowed>no</timeout_allowed>
  </command>

  <!-- Execute kill-webshell script> -->
  <active-response>
    <disabled>no</disabled>
    <command>kill-webshell</command>
    <location>server</location>
    <rules_id>100510</rules_id>
    <timeout>10</timeout>
  </active-response>
```

### Update Configuration

```console
  systemctl restart wazuh-manager
```

## Changes to `local_decoder.xml`

**PATH**: `/var/ossec/etc/decoders`

**FILE**: `local_decoders.xml`

### Add Decoder for Web Shell connections

```xml
  <!-- Decoder for web shell network connection. -->
  <decoder name="network-traffic-child">
    <parent>ossec</parent>
    <prematch offset="after_parent">^output: 'webshell connections':</prematch>
    <regex offset="after_prematch" type="pcre2">(\d+.\d+.\d+.\d+):(\d+)\|(\d+.\d+.\d+.\d+):(\d+)</regex>
    <order>local_ip, local_port, foreign_ip, foreign_port</order>
  </decoder>
```

## Create `webshell_rules.xml`

This file needs to be created. 

**PATH**: `/var/ossec/etc/rules`

**FILE**: `webshell_rules.xml`

### Rules to Add

```xml
  <!-- Linux Rules. -->
  <group name="auditd, linux, webshell,">
    <!-- This rule detects web shell network connections. -->
    <rule id="100521" level="12">
      <if_sid>80700</if_sid>
      <field name="audit.key">webshell_net_connect</field>
      <description>[Network connection via $(audit.exe)]: Possible web shell attack detected</description>
      <mitre>
        <id>TA0011</id>
        <id>T1049</id>
        <id>T1505.003</id>
      </mitre>
    </rule>
  </group>

  <!-- This rule detects network connections from scripts. -->
  <group name="linux, webshell,">
    <rule id="100510" level="12">
      <decoded_as>ossec</decoded_as>
      <match>ossec: output: 'webshell connections'</match>
      <description>[Network connection]: Script attempting network connection on source port: $(local_port) and destina>
      <mitre>
        <id>TA0011</id>
        <id>T1049</id>
        <id>T1505.003</id>
      </mitre>
    </rule>
  </group>
```

### Update Configuration

```console
  systemctl restart wazuh-manager
```
