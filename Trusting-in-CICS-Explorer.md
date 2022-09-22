---
title: 'Trusting, in CICS Explorer'
description: >-
  Security — and whether we trust the services we’re communicating with — are
  important to avoid man-in-the-middle attacks when we’re…
date: '2022-09-21T17:29:00.133Z'
categories: []
keywords: []
slug: /@dave.nice/trusting-in-cics-explorer-338ef5a27d2c
---

Security — and whether we trust the services we’re communicating with — are important to avoid man-in-the-middle attacks when we’re transferring information.

This article walks through the mechanisms for managing TLS trust in CICS Explorer. It’ll be relevant if you get an error similar to this when making a connection:

```
javax.net.ssl.SSLHandshakeException: com.ibm.jsse2.util.h: PKIX path building failed: java.security.cert.CertPathBuilderException: PKIXCertPathBuilderImpl could not build a valid CertPath.; internal cause is:

java.security.cert.CertPathValidatorException: The certificate issued by … is not trusted
```

This error is CICS Explorer telling you that it does not have enough information to trust the server that you’re connecting to.

The article also explains how these issues can be resolved.

### CICS Explorer

CICS Explorer is an Eclipse based tool for managing CICS installations and developing applications to run in them.

As part of this, there are several ways that CICS Explorer communicates:

*   Connections to servers via the Host Connections view  
    … to a CICS server (CMCI or SM Data)  
    … to an FTP or z/OSMF server (function provided by z/OS Explorer)
*   Connections to update sites to update the software
*   Connections to load or import sets of connections (function provided by z/OS Explorer)

Each of these connections can be secured by TLS if necessary, and have different ways of establishing whether we trust the server that we’re connecting to.

### Chains of certificates

TLS certificates are issued in a hierarchical way. A certificate authority (perhaps Symantec or similar) has a root certificate. They sign and issue an intermediate certificate, and using this intermediate certificate they sign and issue an end-user certificate (perhaps for your website).

There’s a chain of trust — if you trust the parents of a certificate then you can be confident that the certificate itself is genuine.

### Truststores

In Java, certificates are imported into keystores. These are password protected and can be used both as sources of authentication certificates (proving who you as a user are) and as sources of trust certificates (proving who the server you’re connecting to is).

By default, the JVM comes with a store of certificates in the file `jre/lib/security/cacerts`. Out of the box, this file includes certificates for the common certificate authorities — if you buy a certificate for a website, the parent certificates will almost certainly be included. Its purpose is to be a keystore to verify trust — so we can call it a truststore.

When you make an https connection, the remote server provides the relevant set of certificates (certificate authority, any child intermediate certificate, down to the end-user certificate for the website) back to the client that made the connection. The client looks at the parent of the provided end-user certificate to see whether it is able to make a chain from the certificate back to a certificate it trusts in its truststore.

### TLS connections from the JVM

When you make an https connection from within Java, for instance when loading exported connections or accessing an Eclipse update site, the JVM connects to the remote server and receives the set of certificates as part of the TLS handshake. The JVM check for trusts against the certificates it has in the `cacerts` file.

By default, there’s no interactive way for a user to resolve a lack of trust for connections made using the JVM classes. The `cacerts` file must be updated manually.

### TLS connections from connections in Host Connections

To provide more control over authentication when making a connection to a CICS or z/OS system for instance via CMCI, CICS Explorer has overridden the default behaviour and provides the capability to set the trust store from within preferences, Explorer > Certificate Management. The trust store in here is initially set to a copy of the JVM’s `cacerts` file, copied into your CICS Explorer workspace. This copy is made the first time you use this new workspace.

If a connection is made from the Host Connections view and the server is not trusted, a pop-up is shown allowing the user to “accept” the certificate into the CICS Explorer workspace copy of the `cacerts` file. This seems very convenient but it’s better not to push a decision about trust down to the end user.

### Resolving trust issues

The most common trust issue will be connecting to a server that has a self-signed certificate, or a certificate produced by an internal certificate authority for your enterprise that isn’t recognised by the JVM.

For these issues, you can let your end users accept certificates ad-hoc when they connect.

However if your enterprise has an internal certificate authority that authors certificates, you can reduce the number of untrusted server decisions your end users must make by altering the package that you distribute to add your certificate authority certificates.

Unlike making CICS and z/OS connections, for issues around loading connections and using internal update sites there is currently no option to accept certificates ad-hoc. For these situations the JVM must be able to find a certificate to trust in the `cacerts` file.

### Updating your cacerts file

**1.** identify the JVM that’s being used to run your copy of CICS Explorer. If you download a zip file assembly containing CICS Explorer, it should be within a `jre` or `jdk` folder inside `IBM Explorer for z/OS`. You can check by opening CICS Explorer and looking at Help > About > Installation Details. Choose the Configuration tab and look for a `java.home` line.

**2.** locate the cacerts file within this JVM. It will be something like `<java.home path>/lib/security/cacerts`, For instance `C:\Users\username\Downloads\IBM Explorer for zOS\jdk\jre\lib\security\cacerts` .

**3.** download the root and intermediate files for your internal certificate authority. This might require discussion with your infrastructure team. The following steps assume that these files are in `DER` format which often end in `.der` or `.cer`.

**4.** import the root and intermediate certificate, like this:

```
keytool -importcert -alias mycaroot -keystore <path\_to>/jre/lib/security/cacerts -storepass changeit -file carootcert.der  
keytool -importcert -alias mycaint -keystore <path\_to>/jre/lib/security/cacerts -storepass changeit -file caintermediatecert.der
```

Notice that we’ve specified an alias which needs to be unique within the keystore.

Having imported these certificates, you should find that you are able to load connections from or use an update site that came from your internal certificate authority. To use a CMCI connection or similar, you’ll need to create a new workspace to have the `cacerts` copy recreated.

### What if it doesn’t work?

Check the CICS Explorer error log. If you are still getting a PKIX error then potentially you don’t have the right parent certificates installed — try visiting the site in a browser and checking the certificate information there.

If you getting a `Host: <hostname> does not match certificate: <hostname>` error or `No name matching <hostname> found` then you trust the server’s certificate, but it was generated for a hostname that’s different to the hostname you’re using to connect with. You’ll need to update the hostname that you’re using so that it matches the certificate.
