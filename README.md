# Restrict the use of admin policies for roles created for a specified auth mount.

## As a user with an admin policy attached to their token.

To test this policy, let's define an approle auth method as follows. Please note that this will work with other auth method as well.

```
vault auth enable -path=approle-test approle
```

Let's create an ACL policy allowing the creation and management of roles in the auth method.

```
vault policy write test-sentinel -<<EOF
path "auth/approle-test/role/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF
```

Let's create an EGP policy called `restrict-policy` and have it apply for the `auth/approle-test/role/*` path.

```
policy=$(base64 check.sentinel)

vault write sys/policies/egp/restrict-policy \
  policy="${policy}" \
  paths="auth/approle-test/role/*" \
  enforcement_level="hard-mandatory"
```

## As a user with the `test-sentinel` policy (created above) attached to their token.

We can authenticate as a user that gets the `test-sentinel` policy we defined above. Or we can, for the purposes of testing, create a token with this policy attached.

```
vault token create -policy=test-sentinel
```

Let's test the EGP policy we created above by trying to create a role with the `admin` or `administrator` policies attached. Both of these should result in an error.

```
vault write auth/approle-test/role/test-role token_policies="admin"
```

```
vault write auth/approle-test/role/test-role token_policies="zoo,administrator,foo"
```

For example:

```
Error writing data to auth/approle-test/role/test-role: Error making API request.

URL: PUT https://vault.example.com:8200/v1/auth/approle-test/role/test-role
Code: 403. Errors:

* 2 errors occurred:
	* egp standard policy "root/restrict-policy" evaluation resulted in denial.

The specific error was:
<nil>

A trace of the execution for policy "root/restrict-policy" is available:

Result: false

Description: main rule

print() output:

Namespace path: 
Request path: auth/approle-test/role/test-role
Request data: {"token_policies": "admin"}
You may not attach an administrator policy to this role.


Rule "main" (byte offset 817) = false

Rule "precond" (byte offset 19) = true

	* permission denied
```

Let's now create a role with some other policy attached. This should succeed.

````
vault write auth/approle-test/role/test-role token_policies="not-admin"
```

For example:

```
vault write auth/approle-test/role/test-role token_policies="not-admin"
Success! Data written to: auth/approle-test/role/test-role
```

We should also be able to delete the role we created.

```
vault delete auth/approle-test/role/test-role
```

## Reference:

* [Sentinel Policies in Vault](https://www.vaultproject.io/docs/enterprise/sentinel)
* [Vault /sys/policies API](https://www.vaultproject.io/api/system/policies)
* [Example Vault Sentinel Policies](https://www.vaultproject.io/docs/enterprise/sentinel/examples)
* [Properties in Vault Sentinel Policies](https://www.vaultproject.io/docs/enterprise/sentinel/properties)
* [Learn Guide for Sentinel Policies in Vault](https://learn.hashicorp.com/vault/identity-access-management/iam-sentinel)

