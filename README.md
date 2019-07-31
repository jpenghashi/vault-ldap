# LDAP auth in Vault

Some reading - https://www.vaultproject.io/docs/auth/ldap.html

This repo serves as testing ground for vault-ldap integration.

The whole setup can be done in Cloud Shell and I highly recommend using that!!

## Setup

Clone the repo. CD into the `vault-ldap-testing` directory.

Standup ldap and phpLDAPadmin with docker compose.
```
docker-compose up -d
```

Create a sample group on the ldap container. Admin password is 'admin'.
```
docker exec -ti vault-ldap-testing_ldap_1 ldapadd -x -D "cn=admin,dc=example,dc=org" -w admin

##type this on the prompt and do ctrl+d after a new line at the end
dn: cn=dev,dc=example,dc=org
cn: dev
objectClass: posixGroup
gidNumber: 600

Ctrl+D
```

And add a sample user account to the group.

```
docker exec -ti vault-ldap-testing_ldap_1 ldapadd -x -D "cn=admin,dc=example,dc=org" -w admin

##type this on the prompt and do ctrl+d after a new line at the end
dn: cn=testuser,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: posixAccount
sn: user
gidNumber: 600
uidNumber: 1009
uid: 1009
homeDirectory: /home/test
userPassword: test

Ctrl+D
```
Check your LDAP objects with phpLDAPAdmin by logging to http://localhost:8080 or by doing a web preview in your Cloud Shell console.
Use `cn=admin,dc=example,dc=org` ad Login DN and password `admin` to login.

## Integrate with Vault
Download and extract vault binary if you have not done so.
```
wget https://releases.hashicorp.com/vault/1.1.0/vault_1.1.0_linux_amd64.zip
unzip vault_1.1.0_linux_amd64.zip
sudo mv vault /usr/local/bin/
```

Run vault in dev mode and login with the root token generated

```
vault server -dev 2>1 log &
export VAULT_ADDR=http://127.0.0.1:8200
vault login
```

Enable ldap auth. Use admin account as bind DN. Note that `userattr` is tied to `cn`, in AD this should be `samAccountName`.
```
vault auth enable ldap
vault write auth/ldap/config url="ldap://127.0.0.1" \
  userdn="dc=example,dc=org" userattr="cn" \
  groupdn="cn=dev,dc=example,dc=org" \
  starttls="false" \
  binddn="cn=admin,dc=example,dc=org"  bindpass='admin'
```

Grant testuser policies `foo` and `bar`. More reading - https://www.vaultproject.io/docs/concepts/policies.html
```
vault write auth/ldap/users/testuser policies=foo,bar
```

Test if user can login with their account to vault. Type in the ldap password (`test`) to login to vault.
```
vault login -method=ldap username=testuser
```

Now the user is authenticated and now can access secrets assigned in the policies.