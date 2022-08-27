# KeyCloak: Setup

Walking through the steps I take to set up KeyCloak. I've used it as an open source option for experimenting and learning about several technologies including: 

- Single Sign-On (SSO) and Federation
- Security Assertion Markup Language (SAML)
- JSON Web Token (JWT)

Integrating this with a reverse proxy like Apache Knox provides SSO to multiple Hadoop UIs and applications.

---

<!-- toc -->

- [Linux Node Prereqs](#Linux-Node-Prereqs)
- [Starting Keycloak Docker Container](#Starting-Keycloak-Docker-Container)
- [Setting up Docker Connectivity](#Setting-up-Docker-Connectivity)
- [Keycloak Realm Setup](#Keycloak-Realm-Setup)

<!-- tocstop -->

---

## Linux Node Prereqs 


__Set up Networking:__

Set the hostname: 
```
# echo 'HOSTNAME=keycloak.example.com' >> /etc/sysconfig/network
# hostnamectl set-hostname keycloak.example.com
# /etc/init.d/network restart
```

> Note: If DNS is not set up, ensure the /etc/hosts file is populated and synced between the nodes.

__Set up Docker:__

Install the packages
```
yum install yum-utils device-mapper-persistent-data lvm2 -y

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install docker-ce docker-ce-cli containerd.io -y

systemctl start docker
systemctl status docker
systemctl enable docker
```


## Starting Keycloak Docker Container

```
docker run -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=admin -e DB_VENDOR=H2 -p 8080:8080 --name keycloak jboss/keycloak

# Control + C to exit and then run the following to start in background

docker start keycloak
```

Can now access at default url:  `http://host:8080`

Login with default credentials: `admin:admin`

## Setting up Docker Connectivity

Get the Docker container UUID: 

```
docker container ls -a --no-trunc
> CONTAINER ID
> 7dab7c5a09ff915f3dd74d771c2b73348dba517ea717b57ad35decd81eec30c7
```
Add entries to hosts file for container: 
```
cd /var/lib/docker/containers/7dab7c5a09ff915f3dd74d771c2b73348dba517ea717b57ad35decd81eec30c7/
vi hosts
```
> **NOTE**: This will be required to ensure that the container can connct to the external LDAP in the next step. 

## Keycloak Realm Setup
- Add Realm: `knoxsso`
- Clients > Create
  - Client ID: knox-saml
  - Client Protocol: saml
- Save

__Settings:__
- Root Url: `https://Knox_host:8443/`
- Valid Redirects: `https://knox_host:8443/*`
- IDP initiated SSO URL Name: `knox-saml`
- Target IDP initiated SSO URL: `http://keycloak.example.com:8080/auth/realms/KnoxSSO/protocol/saml/clients/knox-saml`

> We provide an asterik (*) for Valid Redirects to allow Knox interactions with 'originalUrl'.

__Metadata files:__
- Click Clients > Installation > Format Option > Mod Auth Mellon files > Download
- Import files to Knox Host: /opt/knox-saml

*Knox Host:*
Make metadata directory and copy idp/sp xml files
```
# mkdir -p /var/lib/knox/gateway/data/saml/metadata
# cp /opt/knox-saml/{idp,sp}-metadata.xml /var/lib/knox/gateway/data/saml/metadata
# chown -R knox:knox /var/lib/knox/gateway/data/saml
```

__Syncing Users:__

We will use an existing LDAP Server to populate the users and groups. See my [openLDAP_Setup](https://github.com/Twizzlerific/openLDAP_Setup) repo for how I created my LDAP Server: 
```
User Federation > Ldap 
Edit Mode: Read_only
Vendor: Other
Username LDAP attribute : uid
RDN LDAP attribute : cn
UUID LDAP attribute : entryUUID
User Object Classes : person
Connection URL : ldap://ldap.example.com
Users DN : ou=Users,ou=Marvel,dc=ks,dc=com
Custom User LDAP Filter : (objectClass=person)
Search Scope : One Level
Bind Type : simple
Bind DN : cn=oneaboveall,dc=ks,dc=com
```

! Check Connection and Authentication

---
DiBtP ("Done is Better than Perfect!")