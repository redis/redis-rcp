RCP  - Add TLS support to Redis
===

```
Author: Madelyn Olson <matolson@amazon.com>
Creation date: 12/06/2017  
Update date: 
Status: draft
Version: 1.0
Implementation: TBD
```

History
---

* Version 1.0 (2017-12-13): Initial version.

Proposal 
---
Add TLS support to Redis so that any communication over the wire is encrypted. This includes communication with Redis clients, replication traffic and cluster bus communication.


Rationale
---

Many users who want to store sensitive data in Redis are not able to use Redis because Redis sends data in plain text over the wire. Enterprise users also need support for encryption for regulatory compliance such as HIPAA and PCI. In absence of TLS support, users either don’t use Redis or use some other work around like and SSL proxy, which have higher maintenance and operational overhead. Including TLS in Redis will give developers peace of mind from increased security and enable enterprise users to deploy Redis in user-facing and mission critical applications.

Commands introduced
---
New Startup Parameters: 
`enable-tls`: Turns the TLS feature on. Valid values are “yes” or “no”. Default value is “no”. If set to “yes”, need three additional parameters - certificate-file, private-key-file, dh-params-file. The dh parameters are not dynamically configurable once launched.
`tls-mode`: An optional parameter provided to indicate the level Redis should verify user TLS connections. Setting the value to “require” will make sure all connections are protected with TLS but will not validate the CA or the authenticity of the Redis host. Setting the value to “verify-ca” will validate the CA is trusted, but not verify the authenticity of the host. “verify-full” will validate that both the CA is trusted, and that the hostname matches the hostname in the certificate. The default will be “verify-full”.  This parameter is dynamically configurable once launched.
`certificate-key-pair`: Path to the certificate and private key files for TLS (PEM encoded certificate)
`dh-params-file`: Path to Diffie-Hellman parameters file (PEM encoded)
`mlock`: An optional parameter to enable or disable s2n from using mlock to prevent cryptographic information from being written to swap. Defaults to yes. 

Certificate Renewal commands
`config set certificate-key-pair “<path-to-certificate> <path-to-certificate-private-key>”


Implementation details
---
We will use S2N library for TLS support. We chose to use S2N over other TLS libraries due to enhanced security it offers. (https://github.com/awslabs/s2n ) S2N has a small and auditable code base, undergoes regular static analysis, fuzz-testing and penetration testing, includes positive and negative unit tests and end-to-end test cases, encrypts or erases plaintext data as quickly as possible, protects data from being swapped to disk or appearing in core dumps, and avoids implementing rarely 
used options and extensions. These features make it a much secure offering compared to alternatives and hence we will use it as TLS library.

* In implementation, our goal is that TLS functionality is isolated to a separate C file/header file (tls.c/tls.h) and rest of the source code has almost zero/very minimal code changes
* We will define 2 methods tlsread and tlswrite in tls.c. If TLS is enabled, these methods will read/write to a TLS connection otherwise these will read/write on a network socket. These two methods will be called by zread and zwrite which will be #defined to read/write when compiled with tls enabled. 
* In source code, we will replace references to read/write system calls where they are sending/receiving data over the wire to zread/zwrite. Also in code, where new connections are being created/accepted, we will use utility methods in tls.c to initialize corresponding TLS connections. We will have a global map to store socket file descriptors to TLS connections.
* Using above approach will ensure that we can keep the TLS functionality mostly isolated to a separate file and transparent to rest of the source code. This way we can seamlessly support TLS for client support, replication and cluster bus communication. 

Certificate Renewals
---
Certificate Authorities usually issue certificates for a duration of a year. Renewing certificates can be a disruptive action as application usually need to be restarted. We will implement certificate renewals in Redis that won’t require any restart. We will provide a config set command to renew certificate. Once this is invoked, Redis will start using the new certificate for any new connections from that point onwards. Already established connections stay intact. We will enforce the constraint that at any point in time the redis server could be using at most 2 differ certificates, the previous certificate and the new certificate. Attempting to set a new certificate file while the previous one is being used will result in all connections using the previous certificate to be dropped.  E.g. if Redis is using cert-v1, and we renewed certificate to cert-v2, and then again renewed to cert-v3, then all client connections using cert-v1 would be disconnected. We believe this approach will satisfy all use-cases as certificates are renewed at interval of a year, keep the complexity low and provide seamless certificate renewals for users.

Frame-based reads/writes
---
SSL reads are frame-based, so a consumer of tlsread may call it with a buffer size that does not consume the entire frame. In this case, s2n will buffer the remainder of the decrypted data until the next call to tlsread. The kernel will not notify Redis that more data is available to read from the socket as in a normal file event. It is possible, therefore, that no new trigger will cause redis to re-read data from the buffer. To solve this, we will implement a new type of event that can be queued if the entire buffer was not consumed. A separate part of the event loop will then drain these connections of their remaining data. 

Similarly, for writes, S2N will encrypt the input buffer in chunks and will not acknowledge any bytes sent until the frame is transmitted to the kernel.  In the event that the kernel is not able to consume the entire buffer, S2N requires the caller to repeat the write call with the same buffer.  This is true in most places in Redis except for best-effort newline pings, which we will handle separately.

TLS certificate and hostname validation in replicas and cluster bus clients
---
Replicas and cluster bus nodes will be acting as TLS clients in our setup. As part of TLS protocol, TLS clients should do certificate validation and hostname validation. For certificate and hostname validation, we will use X509 functionality within OpenSSL library. We will check that certificates are issued by a Trusted CA, and check integrity of the whole certificate chain. Self-signed certificates will be rejected by default but can be enabled for testing via the redis parameter “tls-mode”. We will include a new field in the cluster bus protocol for the host DNS name, which will be used for host validation. This protocol change will make it backwards incompatible with Redis engines not compiled with TLS.

Build
----
* TLS support is optional at build time. Redis can be built without TLS support as well. This will be controlled by a build time flag. Users who are not using Redis with TLS therefore do not need to worry about TLS related dependencies or possible performance impact of the extra code.
* For TLS support, Redis would have a compile time and runtime dependency on S2N and OpenSSL libraries. Currently Redis dependencies reside in its source code. We will have placeholder (empty) folders for S2N and OpenSSL dependencies in deps folder and provide scripts for users to download and build these dependencies. 
