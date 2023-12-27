# Ironpatchable 

- VM OVA file with rules: [Download](https://drive.google.com/file/d/1CblbP695JO71nXXDUNqe4guKqxAjM4VL/view?usp=drive_link).
- VM OVA file without rules: [Download](https://drive.google.com/file/d/1s1PmZAvM1yk_-cFP3UOKQrt5XCcuir3A/view?usp=drive_link).
- THM Room: [Ironpatchable](https://tryhackme.com/jr/ironpatchable).

# Ironpatchable Write-Up

This room will require a bit of research to complete it. After all, it's exam time!

For convenience, instead of launching all the commands with `sudo`, we will switch to `root`.

```console
su - root
```

**Password:** ironhack

## Pre-requisites

**NOTE:** This write-up will not repeat several steps described in detail in the Ironhackable write-up. Therefore, it is highly recommended to take a look at it first or consider it for steps related to the Ironhackable room.

## Audit

The first part is about implementing auditing rules.

Wazuh has pretty good documentation, so why not check it out: [Monitoring file and directory access](https://documentation.wazuh.com/current/user-manual/capabilities/system-calls-monitoring/use-cases/monitoring-file-and-directory-access.html).

#### Let's sum it up

This command should help in creating alerts based on write actions.

```console
echo "-w /home -p w -k audit-wazuh-w" >> /etc/audit/audit.rules
````

And, we will also need these couple of commands to persist the audit rule.

```console
auditctl -R /etc/audit/audit.rules
auditctl -l
```

**NOTE:** If the audit rules are added in the way described above, the changes will be lost whenever the Audit service is restarted.

### Audit Rule Implementation

So, we can also persist the changes differently, and here's what we will actually do:

1. Add the rules to `/etc/audit/rules.d/audit.rules`.
2. Restart the audit service.

Add the rule to the `audit.rules` file:

```console
echo "-w /var/www/html/uploads -p w -k audit-wazuh-w" >> /etc/audit/rules.d/audit.rules
```

Restart the Audit service:

```console
systemctl restart auditd
```

#### Testing

To verify whether the audit rule was added or not, you can check the `audit.rules` file located at `/etc/audit/`.

```console
cat /etc/audit/audit.rules
```

![Audit Rule Check](images/audit_rule_check.png?raw=true "Audit Rule Check")

If it was added correctly, it will be displayed like in the screenshot above.

Now it's time to test if a custom alert will be displayed in the Wazuh Alerts dashboard. To achieve that, we just need to upload a file in the Ironhackable `/uploads` page, through the web form.

The steps to achieve that are as follows:

1. Go to [http://MACHINE_IP/de/en/application.php](http://MACHINE_IP/de/en/application.php).
2. Select a file you want to upload.
3. Check "Submit Query."
4. Double-check that the file was uploaded correctly by visiting the URL [http://MACHINE_IP/uploads](http://MACHINE_IP/uploads).
5. Check the Wazuh Alerts dashboard to verify the custom audit alert was generated.

#### Result

![Audit Rule Result](images/audit_rule_result.png?raw=true "Audit Rule Result")

### Question and Answers

**Q:** What is the correct path of the audit log file on the server?\
**A:** /var/log/audit/audit.log.

**Q:** Which audit operations is triggered when a file is uploaded on the server?\
**A:** write.

**Q:** Which default Wazuh audit rule shall we use to serve the purpose of the question above?\
**A:** audit-wazuh-w.

**Q:** What is the alert rule ID displayed on the Wazuh Security Alerts dashboard after you successfully implemented the audit rule?\
**A:** 1337.

## Custom Rule

Alright, as hinted in the THM room, in order to implement the custom rules, we will follow the following guide: [Web shell attack detection with Wazuh](https://wazuh.com/blog/web-shell-attack-detection-with-wazuh/).

Three principal steps are required:

1. Add Auditd rules.
2. Add the XML block to `ossec.conf` file.
3. Add the decoder for web shell connections in the `local_decoder.xml` file.
4. Create a new file `webshell_rules.xml`.

### Audit Changes

The correct process will be to:

1. Add Auditd rules that detect command execution from user www-data.
2. Add Auditd rules that detect network connections from user www-data.
3. Restart the Auditd service.

#### Add Auditd rules that detect command execution from user www-data.

```console
echo "-a always,exit -F arch=b32 -S execve -F uid=33 -F key=webshell_command_exec" >> /etc/audit/rules.d/audit.rules
echo "-a always,exit -F arch=b64 -S execve -F uid=33 -F key=webshell_command_exec" >> /etc/audit/rules.d/audit.rules
```

#### Auditd rules that detect network connections from user www-data.

```console
echo "-a always,exit -F arch=b64 -S socket -F a0=10 -F euid=33 -k webshell_net_connect" >> /etc/audit/rules.d/audit.rules
echo "-a always,exit -F arch=b64 -S socket -F a0=2 -F euid=33 -k webshell_net_connect" >> /etc/audit/rules.d/audit.rules
echo "-a always,exit -F arch=b32 -S socket -F a0=10 -F euid=33 -k webshell_net_connect" >> /etc/audit/rules.d/audit.rules
echo "-a always,exit -F arch=b32 -S socket -F a0=2 -F euid=33 -k webshell_net_connect" >> /etc/audit/rules.d/audit.rules
```

#### Restart Auditd service

```console
systemctl restart auditd
```

#### Testing

Like before, double-check that the rules were added correctly by executing:

```console
cat /etc/audit/audit.rules
```

### Add Changes to `ossec.conf` File

Add the following block of XML to the `/var/ossec/etc/ossec.conf` file. Obviously, use your text editor of choice within the server. For example:

```console
nano /var/ossec/etc/ossec.conf
```

```xml
<!-- Detect webshell connections by checking open TCP connections -->
<localfile>
    <log_format>full_command</log_format>
    <command>ss -nputw | egrep '"sh"|"bash"|"csh"|"ksh"|"zsh"' | awk '{ print $5 "|" $6 }'</command>
    <alias>webshell connections</alias>
    <frequency>120</frequency>
</localfile>
```

### Add Changes to `local_decoder.xml` File

Add the following block of XML to the `/var/ossec/etc/decoders/local_decoder.xml` file.

```console
nano /var/ossec/etc/decoders/local_decoder.xml
```

```xml
<!-- Decoder for web shell network connection. -->
<decoder name="network-traffic-child">
  <parent>ossec</parent>
  <prematch offset="after_parent">^output: 'webshell connections':</prematch>
  <regex offset="after_prematch" type="pcre2">(\d+.\d+.\d+.\d+):(\d+)\|(\d+.\d+.\d+.\d+):(\d+)</regex>
  <order>local_ip, local_port, foreign_ip, foreign_port</order>
</decoder>
```

### Create `webshell_rules.xml` File (and add rules)

First, create the `webshell_rules.xml` file:

```console
touch /var/ossec/etc/rules/webshell_rules.xml
```

Afterwards, add the following block of XML:

```console
nano /var/ossec/etc/rules/webshell_rules.xml
````

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

<group name="linux, webshell,">
  <rule id="100510" level="12">
    <decoded_as>ossec</decoded_as>
    <match>ossec: output: 'webshell connections'</match>
    <description>[Network connection]: Script attempting network connection on source port: $(local_port) and destination port: $(foreign_port)</description>
    <mitre>
      <id>TA0011</id>
      <id>T1049</id>
      <id>T1505.003</id>
    </mitre>
  </rule>
</group>
```

Finally, to apply all changes, the `wazuh-manager` service needs to be restarted:

```console
systemctl restart wazuh-manager
```

### Testing

To fire the new custom alerts, a web shell connection needs to be established:

1. Upload a PHP reverse shell script using the Ironhackable web form.
2. Start a netcat listener locally.
3. Execute the PHP reverse shell script by clicking on it in the `uploads` directory through a browser.

#### Result

![Custom Web Shell Alerts](images/custom_web_shell_alerts.png?raw=true "Custom Web Shell Alerts")

### Question and Answers

**Q:** In which file should be added the XML block to detect open web shell TCP connections?\
**A:** /var/ossec/etc/ossec.conf.

**Q:** In which file should be added the XML block related to the web shell network connections decoder?\
**A:** /var/ossec/etc/decoders/local_decoder.xml.

**Q:** Which file related to the web shell custom linux rules needs to be created?\
**A:** /var/ossec/etc/rules/webshell_rules.xml.

**Q:** In which file should be added the custom linux web shell rules?\
**A:** /var/ossec/etc/rules/webshell_rules.xml.

## Active Response

Once again, we need to rely on the Wazuh documentation. All the details we need can be found here: [How to configure active response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response/how-to-configure.html).

The THM room mentions that we don't need to create a custom script to execute through the Active Response. Instead, we should use the `kill_webshell` script that is already available in the `ossec.conf`.

All we need to do then is to add the Active Response XML block to the `ossec.conf`, as mentioned in the documentation. Of course, we slightly need to adapt it:

```console
nano /var/ossec/etc/ossec.conf
```

```xml
<!-- Execute kill-webshell script> -->
<active-response>
  <disabled>no</disabled>
  <command>kill-webshell</command>
  <location>server</location>
  <rules_id>100510</rules_id>
  <timeout>10</timeout>
</active-response>
```

```console
systemctl restart wazuh-manager
```

### Testing

To check if the Active Response is working, just like in the previous step, a web shell connection needs to be established:

1. Upload a PHP reverse shell script using the Ironhackable web form.
2. Start a netcat listener locally.
3. Execute the PHP reverse shell script by clicking on it in the `uploads` directory through a browser.

If the Active Response is working correctly, the web shell connection should be killed, and an alert containing the flag will appear in the Security Alerts dashboard on Wazuh.

#### Result

![Active Reponse Flag](images/active_response_result.png?raw=true "Active Reponse Flag")

### Question and Answers

**Q:** What is the flag?\
**A:** THM{WAZUH_HAS_NO_SECRETS_NO_MORE}.
