---
layout: post   
title:  "Caveats of Network Security Configuration in Android"
date:   2024-06-16 19:39:00 +0200
categories: android security network
---

# Caveats of Network Security Configuration in Android

## Introduction:

Network Security Configuration (NSC) is a powerful tool for Android developers to manage the security of their app's network communications. However, it's not without its quirks and potential pitfalls. 

In this article, we'll delve into the caveats of NSC, its fundamentals, some practical examples, and best practices to ensure your app's network layer is fortified.

## Since when was the Android Network Security Configuration  Property introduced?

Android introduced Network Security Config in API level 24 (Android 7.0 Nougat) to simplify the configuration of security features like trusted Certificate Authorities (CAs), certificate pinning, and cleartext traffic restrictions.

## Certificate Authorities (CAs):

CAs are trusted third-party entities that issue digital certificates. These certificates verify the identity of websites and servers, ensuring that you're communicating with the legitimate entity and not an imposter.

### How it Works:

1. When your app connects to a server over HTTPS, the server presents its certificate.

2. Android's network layer checks if this certificate was issued by a CA that is trusted by the system. This is done checking if this certificate was issued by any of the CA certificates stored on the Android system.

3. If the CA is trusted, the connection proceeds. If not, the connection is rejected.

### How to solve the `CertPathValidatorException: Trust anchor for certification path not found`

This Exception can be thrown in any of the 3 conditions:
1. The CA that issued the server certificate was unknown.
2. The server certificate wasn't signed by a CA, but was self signed.
3. The server configuration is missing an intermediate CA.

For the 1st and 2nd cases, the solution is similar, you will need to include the unknown certificate for the Android framework to be able to validate against, either the app side's stored certificates or on the user's certificates. 

We can see this case frequently on pre-production environments, for which the hosts may use self-signed certificates, and we can include those certificate on the resources/raw folder, to then be include as trust anchors.

For the 3rd case, the reason goes around a [limitation in android](https://developer.android.com/privacy-and-security/security-ssl#:~:text=Desktop%20browsers%20cache,isn%27t%20as%20tolerant.), for which the system doesn't automatically store intermediate certificates found in previous requests. For this reason the solution would need to be on the server side in this 3rd case, and it involves including the intermediary CA in the server chain. You can find more info [here](https://developer.android.com/privacy-and-security/security-ssl#:~:text=To%20fix%20this%20issue%2C%20configure%20the%20server%20to%20include%20the%20intermediary%20CA%20in%20the%20server%20chain.%20Most%20CAs%20provide%20instructions%20on%20how%20to%20do%20this%20for%20common%20web%20servers.).

### Benefits of Relying on CA for Network Security validation:
- Scalability: You only need to trust a handful of well-known CAs, and they take care of verifying thousands of websites.
- Flexibility: If a website changes its certificate (which happens periodically), as long as it's issued by a trusted CA, your app doesn't need to be updated.

### Drawbacks:

- Potential for Compromise: If a CA is compromised, it could issue fake certificates for malicious sites.
- Mis-issuance Risk: CAs can sometimes mistakenly issue certificates to the wrong entities.

## Alternative: Certificate Pinning

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

1. How to Configure Custom CA signed Certificates for Debug builds, to include also debug certificate authorities:

```xml
<network-security-config>
    <debug-overrides>
        <trust-anchors>
            <certificates src="@raw/debug_cas"/>
            <certificates src="system"/>
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

This configuration allows your app to trust those self signed certificates which may be used in pre-production environments only in debug builds, making it easier to enable your development environments.

2. Certificate Pinning:

```xml
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">yourserver.com</domain>
        <pin-set expiration="2025-12-31">
            <pin digest="SHA-256">...</pin> 
        </pin-set>
    </domain-config>
</network-security-config>
```

This pins your app to a specific certificate for release builds, preventing it from trusting potentially malicious certificates presented by attackers attempting Man-in-the-Middle (MitM) attacks.

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

The Network Security Configuration is a powerful ally in securing your Android app's network communications. Understanding its limitations and following best practices will help you create robust and secure applications.

It's important to distingish between the validation of Certificate Authorities and the Certificate Pinning. The CA validation is enabled by default in the Android framework, whenever we make a request to the internet it retrieves the certificate for the https url we are requesting, and then the network layer validates that the Certificate Autority (CA) used to sign that certificate is the same as one of the CAs stored in the system.

It is relatively common that the CA validation needs to be configured for debug builds, because commonly the debug apis contain self-signed certificates, for example your pre-production environments may be pointing to a different host that uses a self-signed certificate for the HTTPS Urls. In that case we can consider either (1) including the certificates on the resources/raw folder and reference as trust anchors for debug builds (as mentioned above), or (2) disable clear text traffic on debug builds only.

Certificate Pinning is a more secure option, compared to CA validation, because the Android framework (i.e through the network security config we provide) verifies the certificates given by the server contain a subset of the certificates we have provided through the NSC, either by their pins, hashes, or even against the whole certificate. Think of it like verifying someone's ID. CA validation is like checking if the ID was issued by a legitimate government agency (e.g., checking the seal and signature). Certificate pinning is like memorizing the person's photo on the ID and then comparing it to the person in front of you to make sure it's the same person. 