# Multi-IdP API

This is a [proposal](https://fedidcg.github.io/charter#proposals) of the [FedID CG](https://fedidcg.github.io/) to extend FedCM to support multiple identity providers to be used.


https://github.com/user-attachments/assets/1604074c-b1db-41d1-a1e3-081589d3f53c


# Proposal Stage: 

This Proposal is at [Stage 1](https://github.com/w3c-fedid/Administration/blob/main/proposals-CG-WG.md#stage-1): Incubation

# Champions

* @npm1

# Participate
- https://github.com/w3c-fedid/multi-idp/issues

# Introduction

The FedCM API currently allows only one identity provider (IDP) to be used at a time.
However, this is limiting to relying parties (RPs) which accept accounts from more than one IDP.
Because the FedCM API can only be invoked once per top-level page, an RP currently needs to pick which IDP to use FedCM with.
This results in the RP surfacing only some of the accounts from the user and missing users which could have otherwise used FedCM had all IDPs been presented to them.
Therefore, this results in adoption barriers for FedCM because RPs do not want to limit themselves to one IDP.
It also results in lower conversion rates because users are not always given the choice to use federation, or they are given the choice but not with an account they would be willing to use.

# Proposal

We propose allowing a single FedCM invocation to allow specifying multiple IDPs at the same time.
This way, an RP which knows all of the IDPs that they support can enumerate the supported IDPs in this invocation.
In addition, if the RP has a preference of some IDP over another, they may specify that preference via the order in which they pass the IDPs to the FedCM call.
This would be considered a hint, as the user agent can override that preference if they know that the user has previously used certain accounts for federation in the RP.
The RP JavaScript code would look like the following:

```html
<script>
const cred = await navigator.credentials.get({
  identity: {
    providers: [
      {
        configUrl: "https://idp1.com/FOO.json",
        clientId: "123",
      },
      {
        configUrl: "https://idp2.com/login/BAR.json",
        clientId: "456",
      }
    ]
  }
});
</script>
```


Given this formulation, the user agent would peform FedCM fetches in parallel for all of the provided IDPs, following the same logic as the existing single IDP FedCM API.
Once all of the accounts have been fetched, the user agent can aggregate the information and show UI to the user so that the user can pick an account to login to the RP from among their signed in accounts.
Because the `get()` call may receive a token from one of several IDPs, the RP now needs to know which IDP the token comes from.
Therefore, we propose extending the `IdentityCredential` with a `configURL` that would be set to the config of the IDP which was chosen by the user.

```
partial interface IdentityCredential : Credential {
    // The config URL of the selected identity provider.
    readonly attribute USVString configURL;
};
```

Given the `configURL` and the `token`, the RP can then proceed to process the `token` as before to create an account or sign in the user.
Because the RP needs to send all IDP information in a single JavaScript call, it would be in charge of the logic of when to invoke the API, when to abort it, etc.
Each IDP would provide the `configURL` and the processing logic for the `token`.

We have also received feedback from other browser vendors that supporting multiple IdPs by combining multiple get() calls would be more practical.
This is because IdPs can call FedCM on their SDK (e.g. [Facebook SDK]([url](https://developers.facebook.com/docs/facebook-login/web/)), [Google SDK]([url](https://developers.google.com/identity/gsi/web/guides/display-google-one-tap))) so that relying parties (RPs) which already embed these IdPs’ SDKs do not necessarily need to make any changes.
We have been brainstorming some ideas on how to allow multi-get FedCM dialogs.
That said, we have also received feedback that even a single-get multi-idp solution works for some use-cases.
In particular, for IDPs that are related or owned by the same entity, the single array solution works best since it is easy to combine the information in this scenario.
But there are a lot of cases where multiple competing IDPs may want to be shown in the FedCM UI.
This scenario is not well solved by the current proposal because it leaves a lot of the heavy weight to the RP, who may not have the resources or expertise to significantly modify the federation logic.
Here is a sample mock of what we want to achieve in the long term:

![image](https://github.com/user-attachments/assets/c9836dab-1dea-4e40-be34-4702ae774523)

# Alternatives Considered

The main alternative considered was whether we begin by allowing combining multiple separate FedCM calls into a single UI.
However, we quickly found that this has a lot of open problems which are hard to solve.
For instance, we'd need to resolve how to handle aborts: what happens if one IDP aborts its call?
Additionally, the RP preference for ordering is not trivially solved in this case: would we need a separate method for the RP to specify its preference?
And last but certainly not least, we'd need to stop receiving IDPs at some point in time or use dynamic UI.
We performed some experiments that showed that waiting until onload to know when all IDPs have been received would result in a very costly delay to the existing single-IDP FedCM calls.
But if we go with dynamically updated UI, we'd need to find a way to show the accounts to the user in a way that is not jarring and does not result in a poor user experience.
