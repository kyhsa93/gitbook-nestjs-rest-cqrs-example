# API Reference

{% api-method method="post" host="" path="https://example.com/accounts" %}
{% api-method-summary %}
Open Account
{% endapi-method-summary %}

{% api-method-description %}
This endpoint allows you to get free cakes.
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

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="post" host="" path="https://example.com/accounts/remittance" %}
{% api-method-summary %}
Remittance
{% endapi-method-summary %}

{% api-method-description %}

{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-body-parameters %}
{% api-method-parameter name="senderId" type="string" required=true %}
sender account id
{% endapi-method-parameter %}

{% api-method-parameter name="receiverId" type="string" required=true %}
receiver account id
{% endapi-method-parameter %}

{% api-method-parameter name="password" type="string" required=true %}
sender account password
{% endapi-method-parameter %}

{% api-method-parameter name="amount" type="string" required=true %}
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

{% api-method method="put" host="" path="https://example.com/accounts/:accountId" %}
{% api-method-summary %}
Update Account
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
{% api-method-parameter name="oldPassword" type="string" required=true %}
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

{% endapi-method-response-example-description %}

```

```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

