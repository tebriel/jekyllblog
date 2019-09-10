---
layout: post
title:  "Rundeck with the Okta LDAP Interface"
date:   2019-09-10 13:51:48 -0400
categories: devops rundeck
---

## Overview

At [Pindrop][pindrop] I've brought in the awesome tool [Rundeck][rundeck] to
manage a lot of the day-to-day operational toil I've experienced managing servers in things like the large
ElasticSearch logging cluster, logstash nodes, ECS nodes and more. Rolling out Rundeck into our PCI
Environments lead us to consider security requirements around authentication, which needed to be more than a
flat file with user hashes.

## Requirements

We discussed the security requirements for a Rundeck server in a PCI Zone, and it had the following needs:

1.  Accessible only to IPs from our office or VPN.
1.  Integrated with current user management (i.e. not using yet another process to add/remove users)
1.  Has 2 Factor Authentication

## Tools at our Disposal

Currently we have Active Directory internally and that is synced to [Okta][okta]. Since AD doesn't support
2FA, my original desire of connecting Rundeck Directly to AD was a bust. I did, however, find that Okta has an
[LDAP Interface][okta ldap interface] which is for just this sort of thing. We can use the [Rundeck
LDAP][rundeck ldap configuration] login configuration to authenticate users, so this seemed like a good fit.

As a note, we don't have an Enterprise License, so we weren't able to use [Rundeck SSO][rundeck sso configuration].

## Setting Up the Okta LDAP Interface with Rundeck

Okay, this is the meat of this post. I'm going to be pretty dry and technical here, and will expand where
necessary to help you understand, but most of this is "do this exact thing because nothing else will work".

### Okta Configuration

First, follow the details under [`Enable the LDAP Interface`][okta ldap interface].

Next, create a user that isn't in any groups, has Read-Only Admin priviledges, and has *NO* 2-Factor
configuration. The no 2-factor is super important. This user will be querying the LDAP interface, and will be
doing so as part of Rundeck, so there will be no opportunity to put 2FA tokens in. Keeping it out of all
groups gives it the least ability possible while also letting it query LDAP.

Now, create an Okta-Only group. This is not an active-directory group. Only Okta groups can be queried by the
LDAP interface, so create an okta group named something like `rundeck_admins`, add some number of users to it.

Enable Okta Verify on your Okta account as a possible 2-factor login, ensuring that your policies allow Okta
Verify as well. All users who want to use 2FA and this must have an okta verify token. I have mine generated
by the [Duo][duo] App (we usually use Duo 2FA), it's just another OTP generator using a QR Code as the seed. Okta
allows you to have multiple 2FA sources, so our general guidance is to use Duo except when using Rundeck, and
then to use an Okta Verify token.

#### Test the User

If you don't have lapdsearch installed, go ahead and install it. Making sure this query works will save you
tons of time later! Replace `<org_subdomain>` and `<ldap_username>` as appropriate, enter the password you
generated for the above user and (fingers crossed) it should return a listing of all users in your
organization. If not, go figure out what's wrong with your policy/reach out to Okta support.

```
# <org_subdomain> is your Okta URL part (company name perhaps)
ldapsearch -H ldaps://<org_subdomain>.ldap.okta.com:636 -D "uid=<ldap_username>,ou=users,dc=<org_subdomain>,dc=okta,dc=com" -W -s SUB "objectclass=*" "*"
```

### Rundeck Configuration

This required a lot of trail and error, and I'm giving it to you free because I don't want anyone to ever have
to do this again. Some things below are commented out, that gets the default value, but the comments
afterwords may have value later.

In a file named `/etc/rundeck/jaas-ldap.conf` add the following content:

```
ldap {
    com.dtolabs.rundeck.jetty.jaas.JettyCachingLdapLoginModule required
      debug="true"
      contextFactory="com.sun.jndi.ldap.LdapCtxFactory"
      // How to connect to okta ldap
      providerUrl="ldaps://<org_subdomain>.ldap.okta.com:636"  // <org_subdomain> is your Okta URL part (company name perhaps)
      bindDn="uid=<ldap_username>,ou=users,dc=<org_subdomain>,dc=okta,dc=com"  // <ldap_username> is the user you created in the section above
      bindPassword="<ldap_password>" // <ldap_password> is the password you assigned to the user above
      // authenticationMethod="simple" // Options are simple, DIGEST-MD5 and sasl (we're def not using sasl)
      // Use a 'bind' to log in instead of comparing a hash in the user's attributes, required for Okta
      // Okta does not offer a password hash in the interface, bind is the only option for authentication
      forceBindingLogin="true"
      // Users don't have permission to search ldap, so we must use the root context which does
      // Only Okta Admins have permission to use the interface, so we must use our new user we created
      forceBindingLoginUseRootContextForRoles="true"
      userBaseDn="dc=<org_subdomain>,dc=okta,dc=com"
      userRdnAttribute="cn"
      userIdAttribute="uid"
      // userPasswordAttribute="userPassword"  // This isn't exposed by okta, which is why we use forceBindingLogin
      userObjectClass="inetorgperson"
      userEmailAttribute="uid"
      userFirstNameAttribute="givenName"
      userLastNameAttribute="sn"
      // Role settings (this is titled 'groups' in okta speak)
      roleBaseDn="ou=groups,dc=<org_subdomain>,dc=okta,dc=com"
      roleNameAttribute="cn"
      roleMemberAttribute="uniqueMember"
      roleObjectClass="groupofUniqueNames"
      // All our rundeck okta groups are prefixed with rundeck, so in the case of rundeck_admin, a member of
      // this group would have role admin
      rolePrefix="rundeck_"
      // Add these roles to each user that successfully logs in
      supplementalRoles=""
      reportStatistics="true"
      cacheDurationMillis="300000"
      timeoutRead="10000"
      timeoutConnect="20000"
      nestedGroups="false";
};
```

Assuming you're on a debian system and installed Rundeck with the deb package (if not, consult the docs), add
the following to `/etc/default/rundeckd`

```
export RDECK_JVM_OPTS="-Drundeck.jaaslogin=true \\
       -Djava.security.auth.login.config=/etc/rundeck/jaas-ldap.conf \\
       -Dloginmodule.name=ldap"
```

Now start/restart rundeck.

### Logging In

Gather the following things:

1.  Your Okta username (something like jdoe@company.com)
2.  Your Okta password (that seems obvious)
3.  Your Okta Verify 2 Factor Password Generator (seeded with the proper QR Code)

Navigate to your Rundeck instance and enter your username in the username field, in the password field, you'll
need to input both your password and your 2FA as follows `yourpass,123123`. So in this example, your password
was `yourpass` and your Okta Verify token was `123123` at that moment. They must be separated by a `,`.

If you could log in, hurray! Go check your group memberships, if not, go check out the
`/var/log/rundeck/service.log` for some of the longest stack traces you've seen in your life (unless you're a
lifelong Java developer and then I'm sure this is par for the course).


[pindrop]: https://www.pindrop.com
[rundeck]: https://www.rundeck.com/
[duo]: https://duo.com/
[okta]: https://www.okta.com/
[okta ldap interface]: https://help.okta.com/en/prod/Content/Topics/Directory/LDAP_Using_the_LDAP_Interface.htm
[rundeck ldap configuration]: https://docs.rundeck.com/docs/administration/security/authenticating-users.html#ldap
[rundeck sso configuration]: https://docs.rundeck.com/docs/administration/security/single-sign-on.html
