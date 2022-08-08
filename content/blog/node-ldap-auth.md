+++
title= "Authenticate a Node application with LDAP"
date= "2019-05-15"
tags= ["NodeJS", "LDAP"]
socialShare=true
+++

This post demonstrates how to authenticate a user against LDAP.

Let's start by installing [basic-auth](https://www.npmjs.com/package/basic-auth) and [ldapauth-fork](https://www.npmjs.com/package/ldapauth-fork) packages

```bash
npm install ldapauth-fork
npm install basic-auth
```

Steps for implementation;

1. Add packages
2. Create a `ldap` variable with authentication configuration
3. Basic auth should prompt for you username and password. Once user is found, verify the given password by trying to bind the user client with the found LDAP user object and given password.

```javascript
const auth = require("basic-auth");
var LdapAuth = require("ldapauth-fork");

var ldap = new LdapAuth({
  url: "ldap://ldap-url:389",
  bindDN: "uid=rc,ou=AppAccounts,ou=People,ou=Entsys,dc=example.com",
  bindCredentials: "credentials",
  searchBase: "ou=entsys,dc=example.com",
  searchFilter: "(uid={{username}})",
  reconnect: true
});

app.use("/api/admin/", (req, res, next) => {
  const credentials = auth(req);
  if (credentials) {
    ldap.authenticate(credentials.name, credentials.pass, function(err, user) {
      if (err) {
        console.log(err.message);
        return res
          .status("401")
          .set({ "WWW-Authenticate": 'Basic realm="Access Denied"' })
          .end("access denied");
      }
      req.user = user;
      next();
    });
  } else {
    return res
      .status("401")
      .set({ "WWW-Authenticate": 'Basic realm="Access Denied"' })
      .end("access denied");
  }
});
```

Visit [basic-auth](https://www.npmjs.com/package/basic-auth), [ldapauth-fork](https://www.npmjs.com/package/ldapauth-fork) packages for more information on configuration.
