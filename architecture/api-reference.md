# API Reference

{% api-method method="post" host="" path="https://example.com/accounts" %}
{% api-method-summary %}
Open Account
{% endapi-method-summary %}

{% api-method-description %}
create new account
{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-body-parameters %}
{% api-method-parameter name="password" type="string" required=true %}
account password
{% endapi-method-parameter %}

{% api-method-parameter name="name" type="string" required=true %}
account name
{% endapi-method-parameter %}
{% endapi-method-body-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=201 %}
{% api-method-response-example-description %}
The resource was created or the operation not guaranteed to be idempotent succeeded
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=400 %}
{% api-method-response-example-description %}
The request is not valid
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=500 %}
{% api-method-response-example-description %}
An unexpected error occurred inside the server
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="post" host="" path="https://example.com/:accountId/withdraw" %}
{% api-method-summary %}
Withdraw
{% endapi-method-summary %}

{% api-method-description %}
withdraw from account
{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-path-parameters %}
{% api-method-parameter name="accountId" type="string" required=true %}
target account id
{% endapi-method-parameter %}
{% endapi-method-path-parameters %}

{% api-method-body-parameters %}
{% api-method-parameter name="amount" type="integer" required=true %}
withdraw amount
{% endapi-method-parameter %}

{% api-method-parameter name="password" type="string" required=true %}
account password
{% endapi-method-parameter %}
{% endapi-method-body-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=201 %}
{% api-method-response-example-description %}
The request was successful
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=400 %}
{% api-method-response-example-description %}
The request is not valid
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=401 %}
{% api-method-response-example-description %}
Authentication failed \(this server authenticates with password, not token or session\)
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=404 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=422 %}
{% api-method-response-example-description %}
The server understood the request and the syntax is correct, but could not fulfill the request
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=500 %}
{% api-method-response-example-description %}
An unexpected error occurred inside the server
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="post" host="" path="https://example.com/:accountId/deposit" %}
{% api-method-summary %}
Deposit
{% endapi-method-summary %}

{% api-method-description %}

{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-path-parameters %}
{% api-method-parameter name="accountId" type="string" required=true %}
target account id
{% endapi-method-parameter %}
{% endapi-method-path-parameters %}

{% api-method-body-parameters %}
{% api-method-parameter name="amount" type="integer" required=true %}
deposit amount
{% endapi-method-parameter %}

{% api-method-parameter name="password" type="string" required=true %}
account password
{% endapi-method-parameter %}
{% endapi-method-body-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=201 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=400 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=404 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=500 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="post" host="" path="https://example.com/accounts/:accountId/remit" %}
{% api-method-summary %}
Remittance
{% endapi-method-summary %}

{% api-method-description %}

{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-path-parameters %}
{% api-method-parameter name="accountId" type="string" required=true %}
accountId
{% endapi-method-parameter %}
{% endapi-method-path-parameters %}

{% api-method-body-parameters %}
{% api-method-parameter name="receiverId" type="string" required=true %}
receiver account id
{% endapi-method-parameter %}

{% api-method-parameter name="password" type="string" required=true %}
sender account password
{% endapi-method-parameter %}

{% api-method-parameter name="amount" type="integer" required=true %}
remittance amount
{% endapi-method-parameter %}
{% endapi-method-body-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="put" host="" path="https://example.com/accounts/:accountId/password" %}
{% api-method-summary %}
Update account password
{% endapi-method-summary %}

{% api-method-description %}

{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-path-parameters %}
{% api-method-parameter name="accountId" type="string" required=true %}
target account id
{% endapi-method-parameter %}
{% endapi-method-path-parameters %}

{% api-method-body-parameters %}
{% api-method-parameter name="password" type="string" required=true %}
old account password
{% endapi-method-parameter %}

{% api-method-parameter name="newPassword" type="string" required=true %}
new account password
{% endapi-method-parameter %}
{% endapi-method-body-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}
The request was successful
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=400 %}
{% api-method-response-example-description %}
The request is not valid
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=401 %}
{% api-method-response-example-description %}
Authentication failed \(this server authenticates with password, not token or session\)
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=404 %}
{% api-method-response-example-description %}
No accounts found matching the given id
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=500 %}
{% api-method-response-example-description %}
An unexpected error occurred inside the server
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="delete" host="" path="https://example.com/accounts/:accountId" %}
{% api-method-summary %}
Delete Account
{% endapi-method-summary %}

{% api-method-description %}

{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-path-parameters %}
{% api-method-parameter name="accountId" type="string" required=true %}
target account id
{% endapi-method-parameter %}
{% endapi-method-path-parameters %}

{% api-method-query-parameters %}
{% api-method-parameter name="password" type="string" required=true %}
account password
{% endapi-method-parameter %}
{% endapi-method-query-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}
The request was successful
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=400 %}
{% api-method-response-example-description %}
The request is not valid
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=401 %}
{% api-method-response-example-description %}
Authentication failed \(this server authenticates with password, not token or session\)
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=404 %}
{% api-method-response-example-description %}
No accounts found matching the given id
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=422 %}
{% api-method-response-example-description %}
The server understood the request and the syntax is correct, but could not fulfill the request
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=500 %}
{% api-method-response-example-description %}
An unexpected error occurred inside the server
{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="get" host="" path="https://example.com/accounts" %}
{% api-method-summary %}
Get Accounts
{% endapi-method-summary %}

{% api-method-description %}

{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-query-parameters %}
{% api-method-parameter name="names" type="array" required=false %}
account names
{% endapi-method-parameter %}

{% api-method-parameter name="page" type="string" required=false %}
pagination count
{% endapi-method-parameter %}

{% api-method-parameter name="take" type="string" required=false %}
number of accounts
{% endapi-method-parameter %}
{% endapi-method-query-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="get" host="" path="https://example.com/accounts/:accountId" %}
{% api-method-summary %}
Get Account
{% endapi-method-summary %}

{% api-method-description %}

{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-path-parameters %}
{% api-method-parameter name="accountId" type="string" required=true %}
target account id
{% endapi-method-parameter %}
{% endapi-method-path-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=404 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

