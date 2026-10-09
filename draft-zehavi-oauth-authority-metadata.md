---
title: OAuth 2.0 Authority Metadata
abbrev: OAuth Authority Metadata
docname: draft-zehavi-oauth-authority-metadata-latest
category: std
ipr: trust200902
submissiontype: IETF
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
  - OAuth
  - authority metadata
  - protected resource
  - scopes
  - rich authorization requests
  - least privilege
  - AuthZEN

author:
  - fullname: "Yaron Zehavi"
    organization: Raiffeisen Bank International
    email: yaron.zehavi@rbinternational.com

normative:
  RFC2119:
  RFC8174:
  RFC6749:
  RFC8414:
  RFC8707:
  RFC9396:
  RFC9728:

informative:
  RFC7662:
  RFC8693:
  RFC8705:
  RFC9068:
  RFC9449:

  AUTHZEN-ISSUANCE:
    title: AuthZEN Profile for OAuth 2.0 Token Issuance
    target: https://openid.github.io/authzen/authzen-oauth-token-issuance-1_0.html
    author:
      - organization: OpenID Foundation
    date: 2026

  AUTHZEN-EXCHANGE:
    title: AuthZEN Binding for OAuth 2.0 Token Exchange
    target: https://openid.github.io/authzen/authzen-oauth-token-exchange-1_0.html
    author:
      - organization: OpenID Foundation
    date: 2026

--- abstract

While OAuth metadata advertises supported scopes and authorization details types, no metadata describes the **authority** those values represent.

This specification defines authority metadata extensions for OAuth 2.0 Protected Resource Metadata and OAuth 2.0 Authorization Server Metadata.

A protected resource can describe the authority it associates with scopes and RAR types, reusable access requirements profiles, and relationships identifying lower-authority alternatives.

An authorization server can publish a simpler description of its local authority vocabulary.

This metadata can inform authentication and authorization decisions before token issuance and exchange, as well as support authority attenuation.

--- middle

# Introduction

OAuth 2.0 {{RFC6749}} uses scope values to express requested and granted access.
OAuth 2.0 Rich Authorization Requests (RAR) {{RFC9396}} provides the `authorization_details` parameter for structured authorization requests.

OAuth 2.0 Authorization Server Metadata {{RFC8414}} and OAuth 2.0 Protected Resource Metadata {{RFC9728}} enable discovery of information about authorization servers and protected resources. Supported scope values and RAR types can be advertised using `scopes_supported` and `authorization_details_types_supported`.

Supported scope values and RAR types are only identifiers, which do not provide a description of their authority: the operations, resources, or effects they authorizes.

For example, a scope named `payments.manage` might permit reading, creating, executing, or deleting payment instructions. Its name alone does not establish which operations are permitted.

These gaps matter when an authorization server or policy decision point evaluates whether requested authority:

* is permitted by an existing grant or local policy;
* aligns with an approved task, purpose, or Mission;
* is broader than necessary;
* has a lower-authority alternative;
* requires particular token presentation or authentication properties.

This specification defines a common metadata structure to describe authority. It supports both scope values and RAR types.

## Publication Roles

The publication roles are distinct:

* A protected resource publishes the authority semantics it applies when enforcing access.
* An authorization server publishes its local understanding of the authority vocabulary it uses when making issuance decisions.

Protected resource metadata is resource-bound through the `resource` attribute defined by {{RFC9728}}.

Authorization server metadata describes an issuer-local authority vocabulary. It does not establish how a resource enforces an issued authority.

## Scope

This specification defines:

* an `authority_metadata` parameter for protected resource metadata;
* an `authority_metadata` parameter for authorization server metadata;
* a common authority description model for scopes and RAR types;
* reusable protected resource access requirement profiles;
* inline attenuation relationships;
* advisory authority ordering through `family` and `rank`.

This specification does not define:

* a universal authorization policy language;
* a universal vocabulary of application actions;
* a replacement for RAR type definitions or schemas;
* an authorization server policy decision algorithm;
* a new OAuth error or remediation protocol;
* an action approval protocol;
* a transaction or cross-resource outcome evidence protocol;
* a general-purpose translation between scopes and RAR objects.

## Relationship to AuthZEN

The AuthZEN OAuth token issuance profile {{AUTHZEN-ISSUANCE}} and token exchange binding {{AUTHZEN-EXCHANGE}} provide mechanisms for policy evaluation in authorization server issuance and exchange flows.

This document provides semantic information which can be used in those evaluations. For example, a deployment can resolve a requested scope into an authority description before consulting a policy decision point.

This specification does not change those profiles or define an additional AuthZEN request attribute. The transport of resolved metadata to a policy decision point is determined by the applicable profile or deployment.

## Conventions and Terminology

{::boilerplate bcp14-tagged}

The following terms are used:

Authority:
: The operations and access a credential or authorization grant permits,
  including applicable resource and parameter restrictions.

Authority description:
: Metadata describing the authority represented by an OAuth scope or
  authorization details type.

Authority publisher:
: The authorization server or protected resource publishing an authority
  metadata document.

Authority consumer:
: An authorization server, client, policy decision point, gateway, or
  other component that consumes authority metadata.

Access requirement profile:
: A named set of properties a protected resource requires when accepting
  access carrying a described authority.

Attenuation:
: Replacement or restriction of authority with authority that grants no
  greater access.

Authority family:
: A publisher-defined grouping within which authority ranks have a
  common interpretation.

# Metadata Parameter {#metadata-parameter}

This specification defines the optional attribute `authority_metadata` for both OAuth Authorization Server Metadata {{RFC8414}} and OAuth Protected Resource Metadata {{RFC9728}}.

The structure depends on the publication context.

## Protected Resource Metadata

In protected resource metadata, `authority_metadata` contains:

`authorities`:
: REQUIRED. An array of authority entries.

`access_requirement_profiles`:
: OPTIONAL. An object whose attribute names identify reusable access
  requirement profiles and whose attribute values describe those profiles.

The protected resource context is supplied by the enclosing document's `resource` attribute. Authority identifiers, profile names, families, and relationship targets are local to that metadata document.

## Authorization Server Metadata

In authorization server metadata, `authority_metadata` contains:

`authorities`:
: REQUIRED. An array of authorization server authority entries.

Authorization server authority entries describe the publisher's local authority vocabulary. They do not contain resource selectors, access requirement profile references, or attenuation relationships.

## Common Requirements

Within either publication context:

* `authorities` MUST be a JSON array.
* Each authority entry MUST be a JSON object.
* A publisher MUST NOT publish more than one entry with the same
  combination of `kind` and `value` in the same metadata document.
* An empty `authorities` array describes no authorities.
* Omission of an authority entry MUST NOT be interpreted as a statement
  that the authority is unsupported or unrestricted.

This specification permits partial publication. An authority consumer cannot assume that the metadata enumerates every authority accepted or issued by the publisher.

## Examples

The examples use illustrative application vocabularies. They are not standardized action, effect, or risk registries.

### Protected Resource Metadata

The following resource accepts a read scope, a management scope that includes read and update access, and a RAR type for document access.

~~~ json
{
  "resource": "https://documents.example.com",
  "authorization_servers": [
    "https://as.example.com"
  ],
  "scopes_supported": [
    "documents.read",
    "documents.manage"
  ],
  "authority_metadata": {
    "access_requirement_profiles": {
      "standard_access": {
        "token": {
          "max_lifetime": 3600
        }
      },
      "sensitive_access": {
        "token": {
          "max_age": 300,
          "max_lifetime": 600,
          "sender_constrained": true
        },
        "authentication": {
          "acr_values": [
            "urn:example:acr:phishing-resistant"
          ],
          "max_age": 300
        }
      }
    },
    "authorities": [
      {
        "id": "documents.read",
        "kind": "scope",
        "value": "documents.read",
        "description": "Read documents accessible to the subject.",
        "family": "document_access",
        "rank": 10,
        "authority": [
          {
            "resource_type": "urn:example:resource-type:document",
            "actions": [
              "urn:example:action:document:read"
            ],
            "effects": [
              "urn:example:effect:data-disclosure"
            ],
            "risk": "low"
          }
        ],
        "access_requirement_profile": "standard_access"
      },
      {
        "id": "documents.manage",
        "kind": "scope",
        "value": "documents.manage",
        "description": "Read and update documents accessible to the subject.",
        "family": "document_access",
        "rank": 20,
        "authority": [
          {
            "resource_type": "urn:example:resource-type:document",
            "actions": [
              "urn:example:action:document:read",
              "urn:example:action:document:update"
            ],
            "effects": [
              "urn:example:effect:data-disclosure",
              "urn:example:effect:state-change"
            ],
            "risk": "medium"
          }
        ],
        "access_requirement_profile": "sensitive_access",
        "relationships": [
          {
            "relation": "can_be_attenuated_to",
            "target": "documents.read"
          }
        ]
      },
      {
        "id": "document_access.rar",
        "kind": "authorization_details_type",
        "value": "urn:example:authorization-details:document-access",
        "description": "Access to specified documents under type-specific restrictions.",
        "family": "document_access",
        "authority": [
          {
            "resource_type": "urn:example:resource-type:document",
            "actions": [
              "urn:example:action:document:read",
              "urn:example:action:document:update"
            ],
            "effects": [
              "urn:example:effect:data-disclosure",
              "urn:example:effect:state-change"
            ],
            "properties": {
              "semantics": "urn:example:semantics:document-access:v1"
            }
          }
        ],
        "access_requirement_profile": "sensitive_access"
      }
    ]
  }
}
~~~

The RAR entry does not receive a fixed rank in this example because its concrete authority varies with the requested actions and document restrictions.

### Authorization Server Metadata

The following AS publishes its local understanding of profile scopes.

~~~ json
{
  "issuer": "https://as.example.com",
  "scopes_supported": [
    "profile.read",
    "profile.manage"
  ],
  "authority_metadata": {
    "authorities": [
      {
        "kind": "scope",
        "value": "profile.read",
        "description": "Read profile information.",
        "family": "profile_access",
        "rank": 10,
        "authority": [
          {
            "resource_type": "urn:example:resource-type:profile",
            "actions": [
              "urn:example:action:profile:read"
            ],
            "effects": [
              "urn:example:effect:data-disclosure"
            ],
            "risk": "low"
          }
        ]
      },
      {
        "kind": "scope",
        "value": "profile.manage",
        "description": "Read and update profile information.",
        "family": "profile_access",
        "rank": 20,
        "authority": [
          {
            "resource_type": "urn:example:resource-type:profile",
            "actions": [
              "urn:example:action:profile:read",
              "urn:example:action:profile:update"
            ],
            "effects": [
              "urn:example:effect:data-disclosure",
              "urn:example:effect:state-change"
            ],
            "risk": "medium"
          }
        ]
      }
    ]
  }
}
~~~

There is no authority `id`, resource selector, access requirement profile, or attenuation relationship.

### Attenuation Decision

A client requests:

~~~ text
resource=https://documents.example.com
scope=documents.manage
~~~

The resource metadata identifies `documents.read` as an attenuation target.

An AS might determine that a read-only task does not need update authority. Its policy may then issue `documents.read` or decline the request and suggest that the client request `documents.read`.

The metadata identifies the containment relationship. It does not determine which response the AS chooses or establish that read access satisfies the client's task.

# Authority Entries

## Common attributes

Authority entries use the following common attributes.

### kind

REQUIRED. A string identifying the OAuth representation of the authority.

This specification defines:

`scope`:
: The entry describes one OAuth scope value.

`authorization_details_type`:
: The entry describes a RAR authorization details type.

Consumers MUST NOT interpret an unrecognized `kind` as one of the kinds defined here.

### value

REQUIRED. A non-empty string.

For `kind` equal to `scope`, `value` identifies one scope value, not a space-separated collection of scope values.

For `kind` equal to `authorization_details_type`, `value` identifies the value of the RAR object's `type` attribute.

Matching of `kind` and `value` is exact and case-sensitive.

### description

OPTIONAL. A human-readable description of the authority.

Consumers MUST treat this attribute as descriptive text, not executable policy.
Natural-language descriptions do not replace the structured authority description or applicable type semantics.

### family

OPTIONAL. A non-empty string identifying a publisher-defined authority family.

A family identifier has meaning only within the publishing metadata document. The same string in another publisher's metadata does not establish a shared vocabulary.

### rank

OPTIONAL. A non-negative integer providing an advisory ordering within
the specified `family`.

An entry containing `rank` MUST also contain `family`.

Lower values express the publisher's preferred ordering toward lower authority within that family. Equal values do not imply equivalent authority.

Rank does not prove containment, interchangeability, or suitability for a task. See {{ordering}}.

### authority

REQUIRED. A non-empty array of structured authority descriptions.

Each element describes an aspect of the authority represented by the entry.
Multiple elements describe the combined authority of the entry;
they are not alternative choices.

The authority descriptions are defined in {{structured-authority}}.

## Protected Resource attributes

Protected resource authority entries additionally use:

`id`:
: REQUIRED. A non-empty string identifying the entry within this metadata
  document. Every `id` MUST be unique within the document.

`access_requirement_profile`:
: OPTIONAL. A string naming one entry in `access_requirement_profiles`.

`relationships`:
: OPTIONAL. An array of inline relationship objects as defined in {{relationships}}.

A referenced access requirement profile MUST exist in the same `authority_metadata` object.

This document supports one profile reference per authority entry.
Profile inheritance and composition are not defined.

## Authorization Server attributes

Authorization server authority entries are identified by their combination of `kind` and `value`.

An authorization server authority entry MUST NOT contain `id`, `access_requirement_profile`, or `relationships`.

Authorization server authority metadata MUST NOT contain `access_requirement_profiles`.

These restrictions distinguish the AS's local authority descriptions from a protected resource's access enforcement expectations.

# Structured Authority Descriptions {#structured-authority}

Each object in an entry's `authority` array contains:

`resource_type`:
: REQUIRED. A non-empty string identifying the application resource type
  to which the described authority applies.

`actions`:
: REQUIRED. A non-empty array of non-empty strings identifying permitted
  application actions.

`effects`:
: OPTIONAL. An array of non-empty strings describing consequences or
  side effects associated with the authority.

`risk`:
: OPTIONAL. A non-empty string identifying a publisher-defined risk
  classification.

`properties`:
: OPTIONAL. A JSON object containing additional domain-specific semantic
  attributes.

For example:

~~~ json
{
  "resource_type": "urn:example:resource-type:payment",
  "actions": [
    "urn:example:action:payment:read",
    "urn:example:action:payment:execute"
  ],
  "effects": [
    "urn:example:effect:external-transfer"
  ],
  "risk": "high"
}
~~~

This specification defines the structure of these descriptions, not a universal vocabulary for their values.

Publishers SHOULD use stable identifiers and documented vocabularies. URI identifiers are RECOMMENDED when vocabulary values are intended to be understood across independently operated systems.

A consumer MUST NOT assume that identical local strings from different publishers have identical semantics.

## Completeness and Interpretation

An authority description identifies the authority semantics known to its publisher. It does not independently establish a complete policy model.

In particular:

* `resource_type` does not identify every resource instance accessible
  under the authority.
* An action name does not, by itself, identify permitted parameter values.
* An effect describes consequences; it does not confer additional
  authority.
* A risk label does not establish an ordering of access rights.
* Missing `properties` do not mean that no restrictions exist.

A consumer performing semantic authorization evaluation needs the applicable vocabulary definitions, RAR type semantics, request values, and local policy.

## RAR Interpretation

For a RAR entry, the authority description applies to the type identified by `value`.

The authority represented by a concrete RAR object depends on that object's parameters and the type-specific rules defined under {{RFC9396}}.

The metadata does not replace those rules or duplicate the type's field schema.

For example, two objects of the same payment type may grant different authority because they identify different accounts, beneficiaries, amounts, or actions.

A consumer MUST NOT determine concrete RAR authority solely from its type-level authority metadata.

# Access Requirement Profiles {#access-profiles}

Access requirement profiles are published only in protected resource metadata.

A profile describes properties the resource requires when accepting a token carrying the associated authority. It does not instruct an authorization server to issue a token or grant that authority.

## Profile Structure

`access_requirement_profiles` is a JSON object keyed by profile name.

Each profile can contain:

`description`:
: OPTIONAL. Human-readable explanatory text.

`token`:
: OPTIONAL. Token presentation requirements.

`authentication`:
: OPTIONAL. End-user authentication requirements.

Profile names are local, case-sensitive identifiers. They do not imply standardized security levels.

For example:

~~~ json
{
  "access_requirement_profiles": {
    "sensitive_access": {
      "description": "Requirements for sensitive account access.",
      "token": {
        "max_age": 300,
        "max_lifetime": 600,
        "sender_constrained": true
      },
      "authentication": {
        "acr_values": [
          "urn:example:acr:phishing-resistant"
        ],
        "max_age": 300
      }
    }
  }
}
~~~

An omitted requirement means that the profile makes no assertion about that property. It does not remove requirements established by other resource policy.

## Token Requirements

The `token` object can contain:

`max_age`:
: OPTIONAL. A positive integer expressing, in seconds, the maximum
  acceptable elapsed time since the access token was issued.

`max_lifetime`:
: OPTIONAL. A positive integer expressing, in seconds, the maximum
  acceptable total lifetime of the access token.

`sender_constrained`:
: OPTIONAL. A boolean. If `true`, presentation must satisfy an
  applicable sender-constraining mechanism.

A boolean value of `false` means that this profile does not require the property. It does not prohibit the property or override stricter policy.

### Sender-Constraining

`sender_constrained: true` means the resource will reject a token that is not sender-constrained.

This attribute does not select a mechanism. Mechanism selection depends on the token profile, applicable discovery metadata, and deployment agreement.

## Authentication Requirements

The `authentication` object can contain:

`acr_values`:
: OPTIONAL. A non-empty array of acceptable authentication context class
  reference values.

`max_age`:
: OPTIONAL. A positive integer expressing, in seconds, the maximum
  acceptable elapsed time since the relevant end-user authentication
  event, represented in the token's introspection response's `auth_time` claim.

If both attributes are present, both requirements apply.

`acr_values` lists alternatives, not cumulative authentication requirements. The resource accepts an authentication context recognized as satisfying at least one listed value.

Authentication freshness is distinct from token freshness. Issuing a new token does not necessarily establish a new authentication event.

## Default Resource Policy

If an authority entry has no `access_requirement_profile`, no additional profile is declared for that entry.

Resource-wide policy and other applicable requirements still apply.

A profile cannot weaken a requirement established by the resource's other enforcement policy.

## Multiple Authorities in a Token

An access token may carry more than one authority.

This specification does not prescribe how an authorization server combines the associated profiles. It may satisfy a common set of requirements, or decline a request according to policy.

At enforcement time, the resource applies the requirements relevant to the authority used for the operation.

If an operation requires multiple authorities, all applicable requirements must be satisfied. A weaker profile cannot cancel a stronger requirement.

# Authority Relationships {#relationships}

Protected resource authority entries can publish inline `relationships`.
Each relationship object contains:

`relation`:
: REQUIRED. A relationship name.

`target`:
: REQUIRED. The `id` of another authority entry in the same metadata
  document.

The containing authority entry is the relationship source. No `from` attribute is used.
This document defines one relationship:

* `can_be_attenuated_to`

The target MUST exist, MUST differ from the source, and MUST be in the same metadata document.

Both entries MUST declare the same `family`.

## can_be_attenuated_to

The relationship:

~~~ json
{
  "relation": "can_be_attenuated_to",
  "target": "documents.read"
}
~~~

asserts that the target represents no greater authority than the source under the protected resource's authority semantics.

This is a containment assertion, not merely a statement that the target is lower risk or belongs to an earlier workflow phase.

For example, if `documents.manage` authorizes both reading and modifying documents, it can declare attenuation to a scope that authorizes reading the same documents.

Conversely, a payment execution authority does not automatically contain payment preparation or payment status access. Those are different operations. A relationship is valid only if the resource's actual semantics establish the asserted containment.

A publisher MUST NOT declare this relationship based solely on rank, risk, or perceived similarity.

The relationship does not assert that the target satisfies the client's task, purpose, or Mission.

## RAR and Cross-Kind Relationships

Relationships can identify targets of a different `kind`.

However, identifying a target does not define a transformation of authorization request data.

For RAR types, authority comparison may depend on concrete object parameters. An asserted relationship does not authorize dropping, widening, or changing parameter restrictions.

A consumer needs applicable type semantics or configured policy to establish a safe concrete transformation. This specification does not define that transformation.

If containment cannot be established for the concrete request, the relationship alone is insufficient grounds for replacement.

## Relationship Interpretation

Containment is transitive when the same resource context and compatible semantic restrictions apply.

A consumer is not required to compute a transitive closure, select a target, or perform substitution.

Unrecognized relationship names MUST NOT be interpreted as `can_be_attenuated_to`.

Consumers traversing relationships SHOULD apply limits to traversal depth and visited entries to avoid unbounded processing.

# Authority Ordering {#ordering}

`family` and `rank` provide advisory ordering.

A consumer can use them to:

* display authorities in a publisher-defined order;
* prioritize evaluation of lower-authority candidates;
* organize authority choices for human review.

Ranks can be compared only within the same family and metadata document.

Rank MUST NOT be used by itself to establish:

* authority containment;
* equivalent authority;
* safe attenuation;
* satisfaction of a purpose or Mission;
* comparable security or risk across publishers.

Where an attenuation relationship and ranks are both published, the target rank SHOULD be no greater than the source rank.

# Authorization Server Use

An authorization server can use this metadata when processing an authorization request, token request, or token exchange request.

A resource indicator {{RFC8707}} can identify the relevant protected resource. The server can obtain its metadata using {{RFC9728}} or use trusted cached metadata or configured information.

Possible uses include:

* resolving scope and RAR authority semantics;
* evaluating authority against a grant or local policy;
* informing an external policy decision point;
* rendering authority descriptions;
* assessing resource access requirements;
* identifying lower-authority alternatives.

This specification does not require a particular processing sequence.

## Publication Is Not an Issuance Decision

An AS-published authority description does not imply that:

* every client can request that authority;
* every subject can grant it;
* the AS will issue it;
* any particular resource accepts it.

Similarly, a resource's published authority description does not authorize issuance by an AS.

Existing grant restrictions, client policy, subject authority, and applicable protocol rules remain in effect.

## Metadata Sources and Conflicts

An authorization server may use local policy or configuration when protected resource metadata is unavailable.

AS-published metadata expresses the AS's local understanding. It does not override the protected resource's enforcement semantics.

If AS and resource descriptions conflict, this specification does not mandate a precedence algorithm or require issuance. The conflict is an input to trust configuration and local policy.

A consumer MUST NOT treat an unresolved semantic conflict as proof of safe attenuation or authority alignment.

## Attenuation Options

Attenuation handling is determined by AS policy.

Possible behaviors include:

* issuing the originally requested authority;
* issuing lower authority where permitted by the applicable OAuth flow;
* declining the original request;
* suggesting a lower-authority request using an applicable remediation mechanism;
* requesting additional consent;
* requiring the client to submit a revised request.

This specification does not require an AS to search relationships, select the lowest rank, or issue an attenuated token.

Where an AS changes granted authority, it remains responsible for the applicable response requirements of {{RFC6749}}, {{RFC9396}}, and the relevant grant or exchange protocol, including {{RFC8693}} where applicable.

No new error code or remediation response is defined here.

# Security Considerations

## Metadata Authenticity

Consumers need to validate metadata using the discovery and security requirements of {{RFC8414}} or {{RFC9728}}, as applicable.

Obtaining a document over TLS is not sufficient if the document's issuer or resource identity does not match the expected identity.

Metadata publication does not independently establish a trusted authorization server and resource relationship.

## Untrusted Resource Locations

An AS that fetches metadata in response to a client-supplied resource indicator can be exposed to server-side request forgery.

Implementations SHOULD validate destinations, constrain redirects, control access to internal addresses, and apply request size and timeout limits.

## Semantic Misrepresentation

A publisher can incorrectly describe a scope, type, or containment relationship.

An authenticated document proves the document's source, not the correctness of its semantic claims.

Consumers should rely only on trusted publishers and understood vocabularies when using metadata for authorization decisions.

## Stale Metadata

Authority semantics and access requirements may change.

Consumers SHOULD use bounded caching and an appropriate refresh strategy.
An old description must not be assumed to remain valid indefinitely.

This specification does not define semantic version negotiation or bind a token to a metadata version.

## Malicious Resource Metadata and Policy Downgrade

A malicious client can select an attacker-controlled resource whose metadata understates the authority represented by a scope or authorization details type, or declares weaker access requirements.

If an authorization server applies that metadata outside its resource context, the client might obtain a token usable at another resource under weaker policy.

An authorization server MUST bind authority metadata used in an issuance decision to the protected resource for which that metadata was validated.
Scope values, authority identifiers, families, and ranks from one resource MUST NOT be used to establish authority semantics for another resource.

An authorization server MUST NOT treat resource metadata as authorization to weaken existing grant restrictions, client policy, subject authority, or AS-local minimum security requirements.

Omitted or weaker access requirements do not override those requirements.

When protected resource authority metadata informs token issuance, the authorization server MUST audience-restrict the resulting token to the resource or resources independently evaluated for that issuance decision, using the mechanisms described in {{RFC8707}}.

A protected resource MUST reject an access token that is not intended for that resource.

An attacker-controlled resource MUST NOT be able to select an audience mapping that makes its tokens acceptable to another resource.

Publication of metadata, including an `authorization_servers` member naming an AS, does not establish that the publisher is trusted by that AS.

Token refresh or exchange that changes the target resource requires evaluation in the new resource context. A previous issuance decision based on another resource's metadata does not establish authority for the new target.

## Requests Naming Multiple Resources

An OAuth request can identify more than one protected resource.
Each resource's authority metadata and access requirements apply only within that resource's context.

An authorization server may issue a token intended for multiple resources, require a revised request, or reject an unsupported combination, subject to the applicable OAuth protocol and local policy.

If the authorization server issues one token intended to satisfy the requirements of multiple resources, that token MUST satisfy all applicable requirements. Requirements that cannot be jointly satisfied MUST NOT be resolved by selecting the weaker requirement.

This specification does not define a profile-merging algorithm.

## Scope Combinations

The authority of several scopes may not equal a simple union of their individual descriptions.

A resource can apply combination-dependent semantics, and a token can also carry claims or authorization details that affect enforcement.

This specification does not define scope combination semantics.
Consumers need applicable resource policy when evaluating combinations.

## RAR Parameter Restrictions

Type-level metadata must not cause a consumer to ignore restrictions in
a concrete authorization details object.

Unsafe transformations can widen document access, payment destinations,
amounts, or other parameter-defined authority.

## Requirements Are Not Enforcement

Publishing sender-constraining or freshness requirements does not implement them.

Resources remain responsible for enforcement.
Issuing a short-lived token does not prove recent authentication.

## Relationship Processing

Relationship graphs can contain cycles or excessive numbers of entries.

Consumers SHOULD bound graph processing. Rank must not be used as a
shortcut for proving containment.

## Consent Rendering

Human-readable descriptions can improve disclosure but do not prove comprehension.

Descriptions SHOULD be displayed as untrusted text. Consumers MUST NOT execute markup, scripts, or instructions embedded in descriptions.

# Privacy Considerations

Authority metadata can disclose supported operations, resource types, and authentication expectations.

Publishers should consider whether public metadata exposes sensitive operational information.

Published profiles SHOULD describe general access requirements rather than user-specific authentication state or individual authorization decisions.

# IANA Considerations

## OAuth Authorization Server Metadata Registration

This specification requests registration in the "OAuth Authorization Server Metadata" registry established by {{RFC8414}}:

* Metadata Name: `authority_metadata`
* Metadata Description: JSON object describing the authorization
  server's local authority vocabulary.
* Change Controller: IETF
* Specification Document: {{metadata-parameter}} of this document

## OAuth Protected Resource Metadata Registration

This specification requests registration in the "OAuth Protected Resource Metadata" registry established by {{RFC9728}}:

* Metadata Name: `authority_metadata`
* Metadata Description: JSON object describing resource-bound authority,
  reusable access requirement profiles, and attenuation relationships.
* Change Controller: IETF
* Specification Document: {{metadata-parameter}} of this document

# Acknowledgments

TODO

--- back

# Editorial Issues

This section is to be removed before publication as an RFC.

* Evaluate whether sender-constraining mechanism identifiers are needed
  in access requirement profiles.

* Evaluate whether additional metadata is needed to associate
  type-specific semantic definitions with RAR entries.

* Determine whether future revisions should support parameter-dependent
  access profiles and attenuation transformations.

* Determine whether extension registries are needed for authority
  kinds, relationship names, and access requirement attributes.

# Document History

-00

* Initial working draft.
* Defined shared scope and RAR authority descriptions.
* Defined protected resource access requirement profiles.
* Defined inline attenuation relationships.
* Defined advisory family and rank ordering.
* Defined simpler AS-local authority metadata without a support flag,
  resource selectors, or authority identifiers.
* Left attenuation response behavior to AS policy.
