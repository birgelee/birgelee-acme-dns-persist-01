---
title: "ACME DNS Persist Validation Method"
abbrev: "ACME dns-persist-01"
category: std

docname: draft-birgelee-acme-dns-persist-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Automated Certificate Management Environment"
keyword:
 - acme
 - DNS
venue:
  group: "Automated Certificate Management Environment"
  type: "Working Group"
  mail: "acme@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/acme/"
  github: "birgelee/birgelee-acme-dns-persist-01"
  latest: "https://birgelee.github.io/birgelee-acme-dns-persist-01/draft-birgelee-acme-dns-persist.html"

author:
 -
    fullname: Henry Birge-Lee
    organization: Princeton University
    email: birgelee@princeton.edu

normative:

  RFC5234:

informative:

  RFC4086:

--- abstract

Domain Control Validation can be performed using a persistent identifier at a well-known DNS prefix. This draft allows for support of this method using the ACME protocol.


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# ACME DNS Persist Validation

DNS Persist Validation proves control of a domain using a persistent identifier placed in DNS.
The ACME server validates control of the domain name by querying DNS to confirm the presence of a particular identifier.

type (required, string):  The string "dns-persist-01"

```
GET /acme/authz/1234/1 HTTP/1.1
Host: example.com
```

```
HTTP/1.1 200 OK
   {
     "type": "dns-persist-01",
     "url": "https://example.com/acme/authz/1234/1",
     "status": "pending"
   }
```

The client constructs the validation domain name by prepending the label "_validation-persist" to the domain name being validated, then provisions a TXT record under that domain.

The value of the TXT record MUST have the following format defined in ABNF as per {{RFC5234}}).

~~~
txt-record-value = *WSP ca-caa-authorization-domain *WSP ";" *WSP [attribute-list] *WSP

attribute-list = (attribute *WSP ";" *WSP attribute-list) / attribute
attribute = attribute-name *WSP "=" *WSP attribute-value

attribute-name = (ALPHA / DIGIT) *( *("-") (ALPHA / DIGIT))
attribute-value = *(value-char / WSP) value-char *(value-char / WSP)
value-char = %x21-3A / %x3C-7E
~~~


In detail:

ca-caa-authorization-domain: This refers to a domain name that the CA is authorized to sign on when seen in a CAA record.

attribute-list: This is a list of attributes which are put after the ca-caa-authorization-domain.

Permitted Attributes and possible values include:


accounturi: This attribute MUST be included in the attribute list. The corresponding value MUST be the account uri associated with the ACME account requesting the certificate.

persistUntil: This attribute is optional. It contains a unix timestamp after which the record can no longer be used for a DCV check. If a CA checks a dns-persist-01 record and its current timestamp is after the persistUntil, the CA MUST NOT consider the validation completed.


Any record that cannot be parsed properly using this ABNF definition, MUST NOT be treated as satisfying the validation method.

For example, if a CA was authorized to sign certificates with a CAA identifier of "example-ca.example" and an ACME client with account uri "https://authority.example/acct/123" was requesting a certificate, the following record would satisfy the dns-persist-01 validation method for the domain "example.com"

```_validation-persist.example.com IN TXT "example-ca.example; accounturi=https://authority.example/acct/123"```


On receiving a response, the server validates the challenge using the following steps.

1.  Queries `_validation-persist.` prepended to the domain beingvalidated in TXT and ensures it retrieves a DNS NoError response withthe requested TXT record set.
2. The CA then enumerates through the TXT records retrieved to ensureat least one of the records meets the following criteria.
  1. The corresponding TXT record adheres to the proper ABNF syntax.
  2. The "ca-caa-authorization-domain" is a domain the CA is permitted to use in CAA records per its CPS.
  3.  The "attribute-list" contains an accounturi attribute and the corresponding value is the accounturi of the ACME client that requested the challenge.
  4.  If persistUntil is contained in the attribute-list, the current unix timestamp is less than or qual to the timestamp contained in the persistUntil field.

The challenge is successful if for at least one TXT record retrieved by the CA, all of the conditions are met. If no DNS record is found, or DNS record and response payload do not pass these checks, then the validation fails.


# Reuse of challenge TXT records

Unlike other ACME challenges, the DNS record used to satisfy "dns-persist-01" does not contain any information that changes on certificate renewal. The intend of the record is that it can be uploaded to DNS once and kept in place to permit all future certificate issuances by authorized ACME clients at authorized CAs. This eliminates the need for an ACME client to have real-time control over critical operational resources associated with a domain while still confirming the intend of the domain owner to authorize a particular certificate request.

Because of this, it is encouraged that ACME clients leave the relevant TXT record(s) used to satisfy this challenge in place between validations. A domain owner may even provision such a record before requesting a certificate from a CA as the entirety of the contents of the record can be determined before the certificate request.

An ACME client SHOULD check for the record's presents itself before instructing the CA to perform the check to reduce load on the CA and provide more detailed feedback to the user if the challenge is configured incorrectly.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
