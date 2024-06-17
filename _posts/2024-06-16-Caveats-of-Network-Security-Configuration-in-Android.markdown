---
layout: post   
title:  "Caveats of Network Security Configuration in Android"
date:   2024-06-16 19:39:00 +0200
categories: android security network
---

# Caveats of Network Security Configuration in Android

## Introduction:

Network Security Configuration (NSC) is a powerful tool for Android developers to manage the security of their app's network communications. However, it's not without its quirks and potential pitfalls. In this article, we'll delve into the caveats of NSC, its history, practical examples, and best practices to ensure your app's network layer is fortified.

## Since When Was This Property Introduced?

Android introduced Network Security Config in API level 24 (Android 7.0 Nougat) to simplify the configuration of security features like trusted Certificate Authorities (CAs), certificate pinning, and cleartext traffic restrictions.

## Certificate Authorities (CAs):

CAs are trusted third-party entities that issue digital certificates. These certificates verify the identity of websites and servers, ensuring that you're communicating with the legitimate entity and not an imposter.

### How it Works:

1. When your app connects to a server over HTTPS, the server presents its certificate.

2. Your app checks if this certificate was issued by a CA that is listed as trusted in your app's Network Security Configuration.

3. If the CA is trusted, the connection proceeds. If not, the connection is rejected.

### Benefits:
- Scalability: You only need to trust a handful of well-known CAs, and they take care of verifying thousands of websites.
- Flexibility: If a website changes its certificate (which happens periodically), as long as it's issued by a trusted CA, your app doesn't need to be updated.

### Drawbacks:

- Potential for Compromise: If a CA is compromised, it could issue fake certificates for malicious sites.
- Mis-issuance Risk: CAs can sometimes mistakenly issue certificates to the wrong entities.

## Certificate Pinning:

Certificate pinning is a more stringent security measure where you specify the exact certificates (their public keys or hashes) that your app should trust for a particular server.

### How it Works:
Similar to CA verification, the server presents its certificate.
Your app compares this certificate against the pinned certificate you've defined in your Network Security 

### Configuration
If there's an exact match, the connection is allowed. Even if the certificate is issued by a trusted CA, if it doesn't match the pinned one, the connection is rejected.

### Benefits of Stronger Security: 
It makes it much harder for an attacker to perform a successful Man-in-the-Middle (MitM) attack, even if they manage to obtain a valid certificate from a trusted CA.

### Drawbacks:
- Inflexibility: If the server changes its certificate, your app will need to be updated with the new pinned certificate to maintain connectivity.
- Maintenance Overhead: You need to carefully manage pinned certificates and update your app if they expire or change.

## Which One to Use?

- CAs: Generally sufficient for most apps, providing a good balance of security and flexibility.
- Pinning: Consider pinning for apps dealing with extremely sensitive data where the risk of a MitM attack is high.

### Common examples on how to use it:

1. Certificate Authorities in Debug Configuration:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" /> 
            <certificates src="system" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

This configuration allows your app to trust user-installed certificates during development, making it easier to test with self-signed or development CAs.

2. Certificate Pinning:

```XML
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">yourserver.com</domain>
        <pin-set expiration="2025-12-31">
            <pin digest="SHA-256">...</pin> 
        </pin-set>
    </domain-config>
</network-security-config>
```

This pins your app to a specific certificate, preventing it from trusting potentially malicious certificates presented by attackers attempting Man-in-the-Middle (MitM) attacks.

### Benefits of Using NSC (vs. Direct Configuration):

Centralized Control: Define security rules in one place, improving maintainability.
Consistent Enforcement: Rules apply consistently across the app's networking code.
Cleartext Prevention: NSC easily disables cleartext (unencrypted) traffic to protect sensitive data.
Debugging Tools: NSC offers debug-overrides to streamline development.
How Is NSC Enforced?

NSC is enforced by the Android framework. When your app initiates a network connection, the system's networking stack consults the NSC file to determine:

1. Which CAs are trusted.
2. If certificate pinning is required.
3. Whether cleartext traffic is allowed.

### Is There a Way to Skip the NSC?

Technically, yes. Developers can bypass NSC by using low-level networking APIs (e.g., directly creating SSLSocket instances) or by ignoring TrustManager validation errors. However, doing so exposes the app to severe security vulnerabilities and is strongly discouraged.

### How Can We Detect Code Not Following NSC in Our Apps?

Tools like Charles Proxy, Wireshark, or Android Studio's Network Profiler can help detect:

1. Cleartext Traffic: Look for HTTP (not HTTPS) traffic to sensitive endpoints.
2. Invalid Certificates: Observe connection failures due to untrusted CAs or pinning violations.
3. Simply looking into the Logcat for  

### Ways to Prevent Using Code That Doesn't Follow NSC:

- Code Reviews: Ensure all network interactions use the recommended Android networking libraries (e.g., HttpsURLConnection, OkHttp) that respect the NSC.
- Static Analysis Tools: Use linters or security scanners to identify code patterns that bypass NSC.
- Dynamic Analysis: Employ tools like Frida or Xposed to monitor your app's network behavior at runtime and detect any NSC violations.
- Security Testing: Regularly conduct penetration testing on your app to uncover vulnerabilities.

### Conclusion:

NSC is a powerful ally in securing your Android app's network communications. Understanding its limitations and following best practices will help you create robust and secure applications.