---
title: "Zone Hopping: A method to prevent zone-walking in DNSSEC"
category: info

docname: draft-fbw-dnsop-dnszonehop-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: DNSOP Working Group
keyword:
 - DNSOp
 - Zone Walking
 - DNSSEC-NSEC
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://datatracker.ietf.org/wg/dnsop/
  github: fatemabw/test-IETF-ID
  latest: https://github.com/fatemabw/test-IETF-ID

author:
 -
    ins: F. Wala
    fullname: Fatema Bannat Wala
    organization: Energy Sciences Network
    email: fatemebw@es.net
 -
    ins: S. Bohacek
    fullname: Stephan K Bohacek
    organization: University of Delaware
    email: bohacek@udel.edu
 -
    ins: N. Buraglio
    fullname: Nick Buraglio
    organization: Energy Sciences Network
    email: buraglio@forwardingplane.net

normative:
  RFC2119:
informative:
  RFC9364:
  RFC4470:
  RFC5155:
  RFC3845:
  IEEE-ZoneHopping:
    target: https://dsn2024uq.github.io/Proceedings/pdfs/DSN-S2024-3LLnxbOIgEGedw7VY2pz6a/957000a104/957000a104.pdf
    title: "Zone-Hopping: Sensitive Information Leakage Prevention for DNSSEC-NSEC"

--- abstract

DNS Security Extension (DNSSEC) as defined by [RFC9364] was developed to address significant security integrity flaws in DNS. Within certain circumstances, information leakage may be possible stemming from a known DNSSEC vulnerability that facilitates a process known as zone walking, which enables the efficient collection of all FQDNs from a given environment. This document describes the problem space as outlined in [IEEE-ZoneHopping] and offers a potential solution, called Zone-Hopping, to aid in addressing the domain information leakage capable via Zone-walking while preserving the integrity of the records for which DNSSEC was originally introduced.

--- middle

# Introduction

DNSSEC introduced the concept of authenticated denial of existence (referred as DoE from now on), where it proved that a negative response actually came from the authoritative server and not from an attacker. DNSSEC uses NSEC [RFC3845] or NSEC3 [RFC5155] records to provide authenticated DoE protection where the domains or the hashes of the domains that encapsulates the queried domain is returned. The need to offer authenticated DoE as part of DNSSEC protocol protection coverage, opened up the protocol to an information leak problem, called zone-walking, in which an adversary can intentionally query non-existent domains and enumerate the complete zone from the owner names disclosed by the NSEC records. There are many tools available to easily enumerate the contents of a signed-zone, the result of which is similar to the result of a zone transfer process, which interestingly is not allowed from external sources.

# Overview
[RFC4470] was introduced, in early 2006, to solve the zone walking problem. This RFC introduced the concept of ”Minimally Covering NSEC Records and DNSSEC Online Signing”, which describes a way to construct DNSSEC NSEC resource records that cover a smaller range of names. According to the RFC:

~~~~~~~~~~
Whenever an NSEC record is needed to prove the non-existence of a name, a new NSEC record is dynamically produced and signed.
The new NSEC record has an owner name lexically before the QNAME (queried name) but lexically following any existing name and a
”next name” lexically following the QNAME but before any existing name.
~~~~~~~~~~

The new NSEC record hence generated would still cover the non-existent query, but with the fake previous-name and the next name, effectively preventing the disclosure of zone contents. There were two adaptations found that were based on [RFC4470] - NSEC3 White lies and Black lies.

As mentioned in RFC 4470 section 5, there are some security risks associated with the methods described above. First, in order to sign NSEC records on-the-fly, the private key needs to be available on the internet-accessible zone’s authoritative servers. Any unintended disclosure of the private key can compromise the whole zone. Second, generating signatures of NSEC records is computationally expensive and makes authoritative servers vulnerable to a denial of service (DoS) attacks. As mentioned, NSEC was added specifically to resolve denial of existence attacks and therefore ensure client’s requested domain is resolved. Ironically, these online signing approaches increase the computational load on the servers and therefore opens an avenue for DoS, which have the same impact as the attack that NSEC was designed to solve.

This drives a need for a practically viable solution that would prevent zone-walking and doesn't expose the servers to DoS attacks or private key compromises. The technique described in this document deals with off-line signing of the records and at the same time provide DoE attack protection for important domains, without compromising on the integrity of the records. This solution is a hybrid approach that combines best of both worlds, i.e. negative responses are handled just like traditional DNS (except for the domains that require DoE protection and hence have corresponding NSEC records), as well as all the records in the zone are signed (have corresponding RRSIGs) to provide integrity of the records classic DNSSEC protection.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Zone Walking: Zone walking is a straightforward process that utilizes NSEC records. The zone walker generates a DNSSEC query for a random FQDN under the desired apex domain. Assuming that the FQDN does not exist, the DNSSEC server will reply with an NSEC record that includes the next existing FQDN and the preceding existing FQDN from the requested FQDN. Here, ”next” and ”preceding” are based on lexicographic order. Therefore, two existing domains have been found. The zone walker then queries for a string that is slightly after the just returned "next" FQDN from the NSEC record, continues following this process until the next existing FQDN is the apex domain (SOA), signaling that there is no next FQDN and that the end of a zone is reached.

DNSSEC: [RFC9364] DNSSEC ensures the integrity of DNS messages, specifically protecting against malicious or forged answers. This is achieved by adding digital signatures to each DNS record hosted by the authoritative DNS server. To this end, DNSSEC introduces some new record types: RRSIG (digital Resource Record SIGnature), DNSKEY (public key record), DS (Delegation Signer, to establish chain of trust), NSEC (Next Secure, proof of nonexistence), NSEC3 (Hashed Next Secure). The signatures use public key cryptography to provide verifiable DNS messages.

NSEC3 White Lies: NSEC3 White Lies, was introduced in 2014 for prevention of zone walking specifically in NSEC3, where fake NSEC3 records are generated on-the-fly that surround the requested name, as described in RFC 4470. The new NSEC3 record will comprise of the hash of the QNAME minus one as owner name and the hash of the QNAME plus one as the next name and therefore existing records that span the NSEC3 hash of the requested QNAME are not disclosed. White lies also requires online-signing of the newly created fake NSEC3 records on the request basis.

Black Lies (BL): Black Lies (BL), described in an RFC Internet draft in 2016, is implemented by utilizing a different secure negative response ”NODATA” instead of ”NXDOMAIN”. When a client queries for a non-existence domain, an NSEC record is dynamically created where the owner name is the same as the QNAME, and the next name as the immediate lexicographic successor of the QNAME (which is generated on-the-fly and doesn’t match any existing domain names in the zone). The response message have RCODE NOERROR, as opposed to NXDOMAIN, since a record matching the QNAME is being returned (of type NSEC). This method requires online-signing, as oppose to the offline signing in NSEC and NSEC3 implementations. This approach provides protection against zone-walking while still providing integrity of FQDN to IP mapping. Consequently, this method is used by several DNS service providers such as Cloudflare and Amazon Route 53.


# Practical Considerations:
[RFC5155] NSEC3 record type was introduced in RFC 5155, that is implemented to prevent clear text retrieval of FQDNs, instead the hashed versions are returned. That wasn't found to be too useful, as most of the hashes suffer collision attacks and since most FQDNs strings are simple and predictable, it became easy to create rainbow tables and do offline cracking of the hashed domains.

[RFC4470] As described in introduction the initial method to prevent zone-walking was introduced in RFC 4470, which has two practical implementations - NSEC3 White lies and Black lies (see the definitions section for the explanation).

# Zone Hopping (ZH)
The objective of this solution is to disable zone walking for specific FQDNs, while still allowing zone walking to find FQDNs that we seek to protect from denial of existence attacks. It is important to note that FQDNs protected from zone walking are not protected from denial of existence attacks. On the other hand, since all signatures are generated offline, this approach does not increase the computational load on the DNS server. Additionally, this method is trivially implemented in BIND9 by simply modifying a configuration file.

The proposed solution modifies the NSEC records for instantiated names, which are pre-generated and signed in advance (off-line), but in a slightly different manner: public and sensitive records are separated in two files and each is signed separately with the same key, thus creating two zone-files. Then, the NSEC records from the sensitive zone file are deleted, and this modified file is merged back into the public zone file. This process yields a final zone file devoid of and NSEC records with sensitive data, effectively preventing their disclosure through zone walking while still serving any requests for the sensitive domains with complete authenticity. This solution offers best of both worlds: sensitive information is handled just like traditional DNS (i.e. like traditional NXDOMAIN negative answers as opposed to specific NSEC), but at the same time, digital signatures (RRSIG) are available to prove the authenticity of the records.

An extreme case of ZH implementation would be just having one NSEC record for SOA pointing to the last available record in the zone, hence, hopping the complete zone and all the domains in between the first and last record. It will effectively prevent zone-walking, i.e. no domain in the zone has NSEC records except the zone apex.

# Exmaple of ZH
To show how ZH is implemented, let's take an example zone called "gotpcap.com". We registered this domain and use it for testing. Mentioned below are the steps involved to implement ZH in BIND9.

First, create a public zone-file with records and signed it using the Zone Signing Key (ZSK). The resulted signed zone is as follows. Note that the records in the zone files shown below are truncated for the better readability.

~~~~~~~~~~
; gotpcap.signed.db.public
; File written on Tue Mar 12 17:52:50 2024
; dnssec_signzone version 9.18.18
; NOT SHOWING SOA AS IT IS SAME IN ALL
a.gotpcap.com.
            604800  IN A    192.168.2.3
            604800  RRSIG   A 13 3 604800
            604800  NSEC    dns.gotpcap.com.
            604800  RRSIG   NSEC 13 3 604800
dns.gotpcap.com.
            604800  IN A    34.125.87.209
            604800  RRSIG   A 13 3 604800
            604800  NSEC    z.gotpcap.com.
            604800  RRSIG   NSEC 13 3 604800
z.gotpcap.com.
            604800  IN A    192.168.2.4
            604800  RRSIG   A 13 3 604800
            604800  NSEC    gotpcap.com.
            604800  RRSIG   NSEC 13 3 604800
~~~~~~~~~~

Next, create a sensitive zone-file with all sensitive records, and again signed it with the same ZSK. The resulted signed zone is as follows.

~~~~~~~~~~
; gotpcap.signed.db.sensitive
; File written on Tue Mar 12 18:06:31 2024
; dnssec_signzone version 9.18.18
; NOT SHOWING SOA Record
b-sensitive.gotpcap.com.
            604800  IN A   10.10.0.1
            604800  RRSIG   A 13 3 604800
            604800  NSEC    c-sensitive.gotpcap.com.
            604800  RRSIG   NSEC 13 3 604800
c-sensitive.gotpcap.com.
            604800  IN A   10.10.0.2
            604800  RRSIG   A 13 3 604800
            604800  NSEC    gotpcap.com.
            604800  RRSIG   NSEC 13 3 604800
~~~~~~~~~~

Then delete all NSEC records from the signed sensitive zone-file and combine the resulting zone-file with the signed public zone-file. The final signed zone is as follows.

~~~~~~~~~~
; gotpcap.signed.db.final
; File written on Tue Mar 12 18:06:31 2024
; dnssec_signzone version 9.18.18
gotpcap.com.
            604800  IN SOA  gotpcap.com.
            604800  RRSIG   SOA 13 2 604800
            604800  NS      dns.gotpcap.com.
            604800  RRSIG   NS 13 2 604800
            604800  NSEC    a.gotpcap.com.
            604800  RRSIG   NSEC 13 2 604800
            604800  DNSKEY  257 3 13 xS9zESD
            604800  RRSIG   DNSKEY 13 2 604800
a.gotpcap.com.
            604800  IN A    192.168.2.3
            604800  RRSIG   A 13 3 604800
            604800  NSEC    dns.gotpcap.com.
            604800  RRSIG   NSEC 13 3 604800
dns.gotpcap.com.
            604800  IN A    34.125.87.209
            604800  RRSIG   A 13 3 604800
            604800  NSEC    z.gotpcap.com.
            604800  RRSIG   NSEC 13 3 604800
b-sensitive.gotpcap.com.
            604800  IN A    10.10.0.1
            604800  RRSIG   A 13 3 604800
c-sensitive.gotpcap.com.
            604800  IN A    10.10.0.2
            604800  RRSIG   A 13 3 604800
z.gotpcap.com.
            604800  IN A    192.168.2.4
            604800  RRSIG   A 13 3 604800
            604800  NSEC    gotpcap.com.
            604800  RRSIG   NSEC 13 3 604800
~~~~~~~~~~

With this final zone-file, the sensitive FQDNs are not longer discovered via zone-walking, as no NSEC records are available for them, but they do have valid signatures in RRSIG records to validate the authenticity of the records. The queries for a non-existent domain between dns.gotpcap.com. and z.gotpcap.com. will get the same NSEC response of:

dns.gotpcap.com. 604800  NSEC    z.gotpcap.com.

However, querying the sensitive FQDNs will generate a valid response from the name-server.

# Security Considerations of ZH
The proposed method of ZH is susceptible to denial of existence attacks for sensitive FQDNs via record replay. However, one could argue that DoE attacks have same impact as denial of service (DoS) attacks. And in reality DoS attack is much more convenient than pulling off a successful DoE, as it doesn't require MiTM of the queries requested by the client to forge a spoofed response that could successfully deny the existence of a valid domain. Hence, we argue that there are far more better ways to DoS a DNS server than trying to DoS an individual resolver/client via a DoE attack.

Combining the solution described above with the other available solutions, we have the following spectrum of solutions that should satisfy most use-cases.
• DNSSEC-NSEC protects against denial of existence attacks, is not especially vulnerable to DoS attack, but is vulnerable to zone walking.
• BL protects against denial of existence attacks and zone walking, but is susceptible to DoS attacks, which have the same impact as denial of existence attacks.
• ZH allows zone walking only on public FQDNs and protects against denial of existence attacks on these public FQDNs. For sensitive FQDNs, zone walking is not possible and denial of existence is possible. Moreover, the ZH is not especially vulnerable to DoS attacks.

## IPv6 Address Discovery
For IPv6 networks, zone-walking is a critical tool for reconnaissance. According to the Office of Management and Budget (OMB) memorandum M-21-07, issued in Nov. 2020, the Federal government plans to transition to IPv6 only networks by 2025. An important benefit of IPv6 is that network reconnaissance through scanning is infeasible since the address space is so large that scanning takes too long unless massive network traffic is generated, which is easily detected. For instance, the private address range is fc00::/7, allowing private addresses to be dispersed randomly within an address space of 2^121 addresses. However, zone walking provides effective way to find all named hosts. Therefore, disabling zone walking will disable a crucial tool employed in malicious IPv6 network reconnaissance.

## ZH vs.Split-view DNSSEC
Zone walking can expose FQDNs from a private network to the public Internet. This vulnerability can be mitigated by using the Split-View DNS architecture, where a private DNS server is used to protect internal domains from being leaked to the public Internet. While this architecture addresses some aspects of the risks associated with zone walking, it does not entirely eliminate them. Specifically, even with a split-view DNS architecture, an external attacker can quickly discover all publicly facing servers and, upon gaining access to the private network, the attacker can efficiently discover internal servers, which facilitates lateral movement within the network, and doesn't really fit into the zero-trust model which explains the blurring line between what is considered non-trusted (External) and trusted (internal) network.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

We would like to acknowledge the contribution of Domain-Tools for providing access to their PDNS database for research purposes, to perform analysis on the real world data collected from their network sensors. We appreciate their help and prompt support.
