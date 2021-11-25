- Title: Scoped API Keys
- Start Date: 2021-10-15
- Specification PR: [#89](https://github.com/meilisearch/specifications/pull/89)
- Discovery Issue: [#51](https://github.com/meilisearch/product/issues/51)

# Scoped API Keys

## 1. Functional Specification

### I. Summary

The SDKs can generate `Scoped API Keys` inheriting from a MeiliSearch's `API key` generated to force filters during a search for an end-user. A `Scoped API Key` is generated on the user server-side code to be used by an end-user. It allows users to have multi-tenant indexes and thus restricts access to documents depending on the end-user making the search request.

### II. Motivation

`Scoped API Keys` are introduced to solve multi-tenant use-case. By managing this, we reduce one of the last major deal-breakers that makes users not choose MeiliSearch as a solution despite all our advantages.

Users regularly request Multi-Tenant indexes. Users today need to set up workarounds to meet this need. Some of them implement reverse-proxy or managed authentication systems like Hasura or Kong to filter what can and cannot be read. Others decide to use server code as a facade to implement the access restriction logic before requesting MeiliSearch. It is difficult to maintain, less efficient, and requires necessary skills that not everyone has.

### III. Glossary

| Term               | Definition |
|--------------------|------------|
| Master Key         | This is the master key that allows the creation of other API keys. The master key is defined by the user when launching MeiliSearch. |
| API Key            | API keys are stored and managed from the endpoint `/keys` by the master key holder. These are the keys used by the technical teams to interact with MeiliSearch at the level of the client code. |
| Scoped API Key     | These keys are not stored and managed by a MeiliSearch instance. They are generated for each end-user by the backend code from a MeiliSearch API Key. They are used by the end-users to only search the documents belonging to them. |
| Multi-Tenancy      | By multi-tenancy, we mean that an end-user only accesses data belonging to him within an index shared with other end-users. |

### IV. Personas

| Persona | Role |
|---------|------|
| Mark    | Mark is a developer for a SaaS company. He will implement the code to communicate with MeiliSearch to solve technical/product needs. |
| UserX   | UserX represents any end-user searching from the frontend interfaces provided by Anna and Mark's company. |

### V. `Scoped API Key` Explanations

#### Summary Key Points

- `Scoped API keys` are generated from a MeiliSearch parent `API key` on the client's server code to resolve multi-tenancy use-case by restricting access to data within an index according to the criteria chosen by the team managing a MeiliSearch instance.
- These `Scoped API keys` can't be less restrictive than a parent `API key` and can only be used for the search action with a predefined forced filter field.
- These `Scoped API keys` are not stored and thus retrievable on MeiliSearch. This is why we highly advise setting an expiration time on that type of API key for security reasons.

#### Solving Multi-Tenancy with `Scoped API Keys`

![](https://i.imgur.com/J4jVe1n.png)

Let's say that `Mark` is a developer for a SaaS platform. He would like to ensure that an end-user can only access his documents at search time. **His database contains many users and he hopes to have many more in the future.**

When a user registers, the backend-side client code generates a `Scoped API Key` specifically for that end-user so he can only access his documents.

The `filter` parameter is set to restrict the search for documents having an `user_id` attribute. This `filter` parameter is contained in the `Scoped API key` payload and cannot be modified during the search by the end-user making the request. `filter` can be made of any valid filters. e.g. `user_id = 10 and category = Romantic`

This `Scoped API Key` is generated from a parent `API key` used to cipher the `Scoped API Key`. On the MeiliSearch side, this permits checking permissions and authorizing the `UserX` search request using this `Scoped API Key`.

---

#### Generating a `Scoped API Key`

```javascript
const scopedApiKeyRestrictions = {
     "indexesPolicy": {
         "products": {
             "filter": "user_id = 1"
         },
         "reviews": {
             "filter": "user_id = 1 AND published = true"
         }
     },
    "expiresIn": 3600
}

export const generateScopedApiKey = () => {
  return (parentApiKey: string, restrictions: scopedApiKeyRestrictions): string => {
    //hash the parentApiKey and keep a prefix of 4 chars of the hashed parentApiKey
    const prefixKey = crypto
        .createHmac('sha256', parentApiKey)
        .digest('hex')
        .substr(0, 4);

    //serialize restrictions (indexesPolicies object and expiresIn)
    const queryParameters = serializeQueryParameters(restrictions);

    //create the secured part (parentApiKey + queryParameters)
    const securedKey = crypto
      .createHmac('sha256', parentApiKey)
      .update(queryParameters)
      .digest('hex');

    //return the generated `Scoped Api Key`
    return Buffer.from(prefixKey + securedKey + queryParameters).toString('base64');
  };
};
```

##### scopedApiKeyRestrictions

The format allows defining specific enforced search filters for accessible indexes (these indexes must be defined in the parent `Api Key` used to generate the `Scoped API key` and have the search action).

If the user does not want to define specific filters for each index accessible to the end-user, he can use the `*` index wildcard rule.

A policy per index allows overriding the `"*"` behavior.

The `scoped API Keys` also accept a number of seconds in the `expiresIn` field until it expires. This field should be mandatory and explicitly set to `null` if no expiration time is needed.


```javascript
const scopedApiKeyRestrictions = {
     "indexesPolicy": {
         "*": { //all search on indexes different than reviews will have the enforced filter `user_id`
             "filter": "user_id = 1"
         },
         "reviews": {
             "filter": "user_id = 1 AND published = true"
         }
     },
    "expiresIn": null //No expiration time ⚠️ Is not recommended for security and quality of life reasons because the only way to revoke it is to delete the parent key
}
```

#### Validity

`Scoped API Keys` expire or are revoked when the parent `API Key` is deleted or expires.

## 3. Future Possibilities

- Handle more search parameters restrictions.
- Extends `Scoped API Keys` to more than `search` action.