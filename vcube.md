# Webex Calling with Certificate-based Local Gateway (CUBE)

This document contains the configuration used for integrating **Cisco CUBE (Catalyst 8000V)** with **Webex Calling** and **Twilio Elastic SIP Trunk** using SIP/TLS and SRTP.

---

## Configuration

```none
##  Baseline configuration
hostname cube1

ip name-server 1.1.1.1
ip domain name example.com

interface GigabitEthernet1
 ip address 10.10.10.5 255.255.255.0

ip route 0.0.0.0 0.0.0.0 10.10.10.1

line vty 0 4
 exec-timeout 30 0
 logging synchronous
 login local
 transport input ssh

ntp server de.pool.ntp.org

## License Configuration

license boot level network-premier addon dna-premier
platform hardware throughput level MB 1000


## Protect STUN Credentials

key config-key password-encrypt PASSWORD
password encryption aes


## Certificate Configuration

!! Create an RSA key pair
crypto key generate rsa general-keys exportable label lgw-key modulus 4096

!! Create a trustpoint for the certificate
crypto pki trustpoint LGW_CERT
 enrollment terminal pem
 fqdn none
 subject-name cn=cube1.example.com
 subject-alt-name cube1.example.com
 revocation-check none
 rsakeypair lgw-key
 hash sha256 

!! Generate CSR
crypto pki enroll LGW_CERT

!! Paste Intermediate X.509 base64 certificate here
crypto pki authenticate LGW_CERT

!! Paste CUBE host certificate here
crypto pki import LGW_CERT certificate

!! Enable TLS 1.2
sip-ua
 crypto signaling default trustpoint LGW_CERT
 transport tcp tls v1.2

!! Install Cisco root CA bundle
ip http client source-interface GigabitEthernet1
crypto pki trustpool import clean url http://www.cisco.com/security/pki/trs/ios_core.p7b

!! Install Digicert global root CA (for Twilio SIP trunk)
crypto pki trustpoint twilio
 enrollment terminal
 revocation-check none

crypto pki authenticate twilio
<paste Digicert global root CA certificate here>


## Global SIP Configuration

voice service voip
 ip address trusted list
  ipv4 23.89.0.0 255.255.0.0  !Webex Calling IPs
  ipv4 62.109.192.0 255.255.192.0
  ipv4 85.119.56.0 255.255.254.0
  ipv4 128.177.14.0 255.255.255.0
  ipv4 128.177.36.0 255.255.255.0
  ipv4 135.84.168.0 255.255.248.0
  ipv4 139.177.64.0 255.255.248.0
  ipv4 139.177.72.0 255.255.254.0
  ipv4 144.196.0.0 255.255.0.0
  ipv4 150.253.128.0 255.255.128.0
  ipv4 163.129.0.0 255.255.128.0
  ipv4 170.72.0.0 255.255.0.0
  ipv4 170.133.128.0 255.255.192.0
  ipv4 185.115.196.0 255.255.252.0
  ipv4 199.19.196.0 255.255.254.0
  ipv4 199.19.199.0 255.255.255.0
  ipv4 199.59.64.0 255.255.248.0
 
  ipv4 54.171.127.192 255.255.255.192 !Twilio Service Provider IPs
  ipv4 54.171.127.192 255.255.255.192
  ipv4 54.244.51.0 255.255.255.0
  ipv4 54.172.60.0 255.255.254.0
  ipv4 54.172.60.0 255.255.255.252
  ipv4 54.244.51.0 255.255.255.252
  ipv4 54.171.127.192 255.255.255.252
  ipv4 35.156.191.128 255.255.255.252
  ipv4 54.65.63.192 255.255.255.252
  ipv4 54.169.127.128 255.255.255.252
  ipv4 54.252.254.64 255.255.255.252
  ipv4 177.71.206.192 255.255.255.252
  exit
 exit

## SIP Global Commands

voice service voip
 address-hiding
 mode border-element
 allow-connections sip to sip
 no supplementary-service sip refer
 trace
 media statistics
 stun
  stun flowdata agent-id 1 boot-count 4
  stun flowdata shared-secret 0 Password123$
  exit
 sip
  asymmetric payload full
  early-offer forced
  sip-profiles inbound
  exit


## SIP Profiles

### Webex Calling (Outbound)

!! SIP profiles for outbound messages to Webex Calling; vCube behind NAT, internal ipv4 10.10.10.5 , NAT public ipv4 192.65.79.20
voice class sip-profiles 100
 rule 10 request ANY sip-header Contact modify "@.*:" "@cube1.example.com:"
 rule 20 response ANY sip-header Contact modify "@.*:" "@cube1.example.com:"
 rule 30 response ANY sdp-header Audio-Attribute modify "(a=candidate:1 1.*) 10.10.10.5" "\1 192.65.79.20"
 rule 31 response ANY sdp-header Audio-Attribute modify "(a=candidate:1 2.*) 10.10.10.5" "\1 192.65.79.20"
 rule 40 response ANY sdp-header Audio-Connection-Info modify "IN IP4 10.10.10.5" "IN IP4 192.65.79.20"
 rule 41 request ANY sdp-header Audio-Connection-Info modify "IN IP4 10.10.10.5" "IN IP4 192.65.79.20"
 rule 50 request ANY sdp-header Connection-Info modify "IN IP4 10.10.10.5" "IN IP4 192.65.79.20"
 rule 51 response ANY sdp-header Connection-Info modify "IN IP4 10.10.10.5" "IN IP4 192.65.79.20"
 rule 60 response ANY sdp-header Session-Owner modify "IN IP4 10.10.10.5" "IN IP4 192.65.79.20"
 rule 61 request ANY sdp-header Session-Owner modify "IN IP4 10.10.10.5" "IN IP4 192.65.79.20"
 rule 70 request ANY sdp-header Audio-Attribute modify "(a=rtcp:.*) 10.10.10.5" "\1 192.65.79.20"
 rule 71 response ANY sdp-header Audio-Attribute modify "(a=rtcp:.*) 10.10.10.5" "\1 192.65.79.20"
 rule 80 request ANY sdp-header Audio-Attribute modify "(a=candidate:1 1.*) 10.10.10.5" "\1 192.65.79.20"
 rule 81 request ANY sdp-header Audio-Attribute modify "(a=candidate:1 2.*) 10.10.10.5" "\1 192.65.79.20"
!

### Webex Calling (Inbound)

voice class sip-profiles 110
 rule 10 response ANY sdp-header Video-Connection-Info modify "192.65.79.20" "10.10.10.5"
 rule 20 response ANY sip-header Contact modify "@.*:" "@cube1.example.com:"
 rule 30 response ANY sdp-header Connection-Info modify "192.65.79.20" "10.10.10.5"
 rule 40 response ANY sdp-header Audio-Connection-Info modify "192.65.79.20" "10.10.10.5"
 rule 50 response ANY sdp-header Session-Owner modify "192.65.79.20" "10.10.10.5"
 rule 60 response ANY sdp-header Audio-Attribute modify "(a=candidate:1 1.*) 192.65.79.20" "\1 10.10.10.5"
 rule 70 response ANY sdp-header Audio-Attribute modify "(a=candidate:1 2.*) 192.65.79.20" "\1 10.10.10.5"
 rule 80 response ANY sdp-header Audio-Attribute modify "(a=rtcp:.*) 192.65.79.20" "\1 10.10.10.5"
!

### SIP Options Keepalive

voice class sip-profiles 115
 rule 10 request OPTIONS sip-header Contact modify "<sip:.*:" "<sip:cube1.example.com:"
 rule 30 request ANY sip-header Via modify "(SIP.*) 10.10.10.5" "\1 192.65.79.20"
 rule 40 response ANY sdp-header Connection-Info modify "10.10.10.5" "192.65.79.20"
 rule 50 response ANY sdp-header Audio-Connection-Info modify "10.10.10.5" "192.65.79.20"
!

### Twilio

voice class sip-profiles 200
 request REINVITE sip-header From modify "(<.*:.*)(@.*>)" "\1@example.pstn.twilio.com>"
 request CANCEL sip-header From modify "(<.*:.*)(@.*>)" "\1@example.pstn.twilio.com>"
 request INVITE sip-header To modify "(<.*:.*)(@.*>)" "\1@example.pstn.twilio.com>"
 request REINVITE sip-header To modify "(<.*:.*)(@.*>)" "\1@example.pstn.twilio.com>"
 request INVITE sip-header From modify "(<.*:.*)(@.*>)" "\1@example.pstn.twilio.com;user=phone>"
 request INVITE sip-header P-Asserted-Identity modify "(<.*:.*)(@.*>)" "\1@example.pstn.twilio.com;user=phone>"
 request ANY sip-header Contact modify "@.*:" "@192.65.79.20:"
 response ANY sip-header Contact modify "@.*:" "@192.65.79.20:"
!

## Local Gateway Tenant (Webex Calling)

voice class tenant 100
  listen-port secure 5062
  no remote-party-id
  sip-server dns:eun01.sipconnect.bcld.webex.com
  srtp-crypto 100
  localhost dns:cube1.example.com
  session transport tcp tls
  no session refresh
  error-passthru
  rel1xx disable
  asserted-id pai
  bind control source-interface GigabitEthernet1
  bind media source-interface GigabitEthernet1
  no pass-thru content custom-sdp
  sip-profiles 100
  sip-profiles 110 inbound
  privacy-policy passthru
!

## Twilio PSTN Tenant

voice class tenant 200
  sip-server dns:example.pstn.twilio.com
  srtp-crypto 200
  localhost dns:cube1.example.com
  session transport tcp tls
  asserted-id pai
  bind control source-interface GigabitEthernet1
  bind media source-interface GigabitEthernet1
  sip-profiles 200
  early-offer forced
!

## Dial-Peers

!
dial-peer voice 100 voip
 description Inbound/Outbound Webex Calling
 destination-pattern BAD.BAD
 session protocol sipv2
 session target sip-server
 session transport tcp tls
 destination dpg 200
 incoming uri request 100
 voice-class codec 100
 voice-class stun-usage 100
 voice-class sip tenant 100
 voice-class sip options-keepalive profile 100
 dtmf-relay rtp-nte
 srtp
 no vad
!
dial-peer voice 200 voip  
 description Inbound/Outbound TWILIO PSTN
 destination-pattern BAD.BAD
 session protocol sipv2
 session target sip-server
 session transport tcp tls
 destination dpg 100
 incoming uri request 200
 voice-class codec 200
 voice-class sip tenant 200
 voice-class sip options-keepalive profile 200
 dtmf-relay rtp-nte
 srtp
 no vad
!
```
