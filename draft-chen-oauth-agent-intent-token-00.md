---
stand_alone: true
ipr: trust200902
cat: std
submissiontype: IETF
area: Security
wg: OAuth Working Group

docname: draft-chen-oauth-agent-intent-token-01

title: The Intent Token - A Credential for Delegated User Intent
abbrev: Intent Token
lang: en
kw:
  - OAuth
  - Intent
  - Agent
  - AI
  - Authorization
  - Token Exchange

author:
- name: Meiling Chen
  org: China Mobile
  city: BeiJing
  country: China
  email: chenmeiling@chinamobile.com
- name: Li Su
  org: China Mobile
  city: BeiJing
  country: China
  email: suli@chinamobile.com
- name: Peter Chunchi Liu
  org: Huawei
  country: China
  email: liuchunchi@huawei.com

normative:
  RFC2119:
  RFC3986:
  RFC6749:
  RFC7519:
  RFC8174:
  RFC8693:

informative:
  RFC7009:
  RFC8705:
  RFC9396:
  draft-chen-oauth-agent-authz-use-cases-04:


--- abstract

In the scenario of AI agent authorization, how to ensure that a user's high-level, natural-language intent is correctly and safely converted into low-level, specific authorization grants. A misunderstanding or malicious interpretation during this translation can lead to significant security vulnerabilities. This document introduces the "Intent Token," a new type of security token designed to bridge this semantic gap. The Intent Token is a signed credential that immutably captures the user's original intent. It decouples the act of granting an intent from the act of issuing specific access tokens. A client presents this Intent Token to an Authorization Server, which acts as a trusted intermediary. The Authorization Server's responsibility is to interpret the user's stated intent and, based on its policies, issue narrowly-scoped, short-lived access tokens for specific resources, thereby ensuring the principle of least privilege is upheld throughout the process.

--- middle

# 1. Introduction

## 1.1. The Semantic Gap in Authorization

Traditional OAuth 2.0 [RFC6749] relies on a pre-defined set of `scope` values that a user consents to. This model works well when the requested permissions are simple and easily understood by the user (e.g., "read your profile," "access your calendar").

However, modern applications, especially those driven by AI and agents, operate based on high-level user goals, not low-level API permissions. A user might command an agent: "Plan my business trip to the IETF meeting in Singapore next month." Such natural language instructions need to be interpreted by AI agents, which then determine the specific resource requests required to complete the task.

For example, to fulfill the abovementioned task, the agent might need to:
*   Read the user's calendar to check for conflicts (`calendar.read`).
*   Access the user's contacts to find travel companions (`contacts.read`).
*   Search for flights and hotels (`flight.search`, `hotel.search`).
*   Book the chosen options using a stored payment method (`payment.execute`).

The core problem is the **semantic gap**: the user approves a high-level *intent*, but the system must translate this into a set of low-level *permissions*. If this translation happens unchecked on the client side, it creates a major security risk. A malicious or buggy client could request far more permissions than the intent strictly requires, and the user would have no way of knowing. This breaks the principle of least privilege. Therefore, authorizing the user's original intent—even if it contains a certain degree of vagueness (such as natural language)—helps verify whether the agent's subsequent resource requests align with that intent, thereby preventing 'intent drift.

It is critical to understand the scope of the cryptographic guarantee provided by the Intent Token's signature. The signature provides a verifiable and non-repudiable proof of the user's original instruction. It does not, however, prove that a client's subsequent interpretation of that instruction is correct. Instead, its purpose is to enable **informed adjudication** by the Authorization Server, which can compare the client's specific request against the user's verified intent.

## 1.2. Core Concept: Separating Grant from Execution

The core value of the Intent Token lies in its introduction of a two-stage model for AI agent-driven authorization, which resolves the "semantic gap" between high-level user intent and specific resource access permissions. This model aligns closely with the conceptual framework proposed in `draft-chen-oauth-agent-authz-use-cases`, which can be understood as the separation of a **Semantic Authorization Stage** and an **Resource Authorization Stage**.

1.  **Semantic Authorization Stage: Establishing Task Intent Baseline**
    *   **User Action**: A user expresses a high-level, business-oriented intent to an AI agent via natural language (e.g., "Help me plan a trip to Paris").
    *   **Core Artifact**: This act of authorization by the user no longer results in the direct issuance of a broad, static set of `scope`s. Instead, it is captured and solidified into a verifiable, signed digital object—the **Intent Token**.
    *   **Conceptual Alignment**: This Intent Token is the concrete manifestation of the **"Semantic Authorization Intent"** that the user bestows upon the agent. It represents the authoritative proof that the user "consents to the agent acting to achieve this goal,". While it helps agents determine required resources and allows the AS to verify approvals, it is not the final access permission itself.

2.  **Resource Authorization Stage: Executing the Intent**
    *   **Agent Action**: When the AI agent, based on its planning (e.g., it first needs to search for flights), determines that a specific operational permission is required (e.g., `scope=flights:search`), it makes a request to the Authorization Server.
    *   **Core Mechanism**: In this request, the AI agent MUST present the **Intent Token** it previously obtained.
    *   **Conceptual Alignment**: At this point, the role of the Intent Token transforms into **"Execution Reference"**. It proves to the Authorization Server that the currently requested, specific, low-level `scope` (`flights:search`) conforms to the user's original, high-level intent ("plan a trip to Paris").

In this way, the Intent Token becomes the critical bridge connecting these two layers:

*   It is created at the **Semantic Authorization Stage** as the carrier of the user's intent authorization, assuring immutability.
*   It is used at the **Resource Authorization Stage** as indispensable reference for converting that intent into concrete resource permissions, avoiding intent drift.

This separation ensures that authorization decisions are no longer a one-time, vague, "all-or-nothing" process. Instead, the Authorization Server can, at each point a specific permission is needed, make a **Just-in-Time**, **Fine-Grained**, and **Secure** decision based on the user's original, immutable intent context provided by the Intent Token. This proposed extension stabilizes the overarching information flow while preserving the flexible selection of agents, thereby balancing the containment of unpredictable agent behaviors with the maximization of their autonomous planning potential.

## 1.3. Authenticating Intent vs. Approving Authority

A critical design principle of the Intent Token is the precise scope of its cryptographic guarantee. The digital signature provides a verifiable and non-repudiable link between the user (sub), the high-level instruction (the intent claim), and the time of approval (iat). It attests to the authenticity and integrity of the original instruction as approved by the user.

Crucially, the signature does not prove that a client's subsequent interpretation of this intent is correct or appropriate. It does not grant the client the authority to be the trusted interpreter of the user's goal. Instead, the Intent Token facilitates a "separation of concerns" in the trust model:

*   The Client (e.g., an AI Agent) acts as a proposer, interpreting the intent and requesting a specific, narrowly-scoped access token to perform a concrete action.
*   The Authorization Server acts as the trusted adjudicator. It receives the signed Intent Token as immutable evidence of the user's original goal. It is the AS's responsibility to inspect this authentic instruction and determine, based on its own policies, whether the client's requested permissions represent a faithful and safe interpretation.
*   
This model allows the Authorization Server to make an informed decision about the client's proposed action, ensuring that the subsequent issuance of access tokens remains securely tethered to the user's original, verified intent.

## 1.4. Distinction from Rich Authorization Requests (RAR) 

While the Intent Token may leverage structures defined by Rich Authorization Requests (RAR, {{RFC9396}}), its architectural role is fundamentally different.

RAR is designed to structure the **"ask"** within a single, synchronous authorization grant. A client uses it to request a rich set of permissions, and if approved, typically receives an immediately usable access token.

The Intent Token, in contrast, represents a long-lived, delegated **"mission"** designed for asynchronous operations. It decouples the moment of user consent from the multiple, future moments of resource access. An agent can hold this token and use it later—when the user is offline—to request specific, just-in-time access tokens under the authority of the original intent.

## 1.5. Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}}.

# 2. Terminology

**Intent**
: A string representing a high-level goal or objective that a user wishes to accomplish. This is typically captured from the user's natural language input or other user-endorsed information.

**Intent Token**
: A security token (e.g., a JWT) that contains the user's intent and associated authorization constraints. It is signed by a trusted entity and serves as a verifiable record of the user's consent to that intent.

**AI Agent (Client)**
: An OAuth 2.0 client that acts on behalf of the user to fulfill a given intent. It uses an Intent Token to request specific access tokens from the Authorization Server.

**Authorization Server (AS)**
: The OAuth 2.0 Authorization Server. In this context, it is enhanced to understand and process Intent Tokens. It validates the Intent Token and uses its claims as crucial input for authorization decisions.

**Resource Server (RS)**
: The OAuth 2.0 Resource Server, which hosts the APIs the AI Agent needs to call. It receives access tokens issued by the AS.

# 3. The Intent Token

## 3.1. Overview

The Intent Token is a signed assertion of a user's original intent. It is designed to be exchanged at an Authorization Server for one or more traditional Access Tokens. It is not intended to be presented directly to a Resource Server.

## 3.2. Format

Intent Tokens are represented as JSON Web Tokens (JWTs) [RFC7519]. They MUST be signed to ensure their integrity and authenticity. The signing algorithm should be negotiated between the token issuer and the Authorization Server.

## 3.3. Claims

An Intent Token contains the following claims:

*   **iss** (Issuer): REQUIRED. As defined in [RFC7519].
*   **sub** (Subject): RECOMMENDED. As defined in [RFC7519]. Represents the user who granted the intent.
*   **aud** (Audience): REQUIRED. As defined in [RFC7519]. The primary audience MUST be the Authorization Server that is expected to process the token.
*   **client_id**(Oauth Client identifier): REQUIRED. The client identifier of the OAuth 2.0 client (the AI Agent) to whom the Intent Token was issued. This binds the token to a specific client.
*   **exp** (Expiration Time): REQUIRED. As defined in [RFC7519]. Intent Tokens may be long-lived, representing a user's standing instruction.
*   **iat** (Issued At): REQUIRED. As defined in [RFC7519].
*   **jti** (JWT ID): RECOMMENDED. As defined in [RFC7519]. A unique identifier for the token, which can be used to prevent reuse.
*   **cnf** (Confirmation): RECOMMENDED. The Confirmation claim, as defined in RFC 7800. It provides a mechanism for sender-constraining the token, binding it to a specific client key. 
*   **authorization_details**: OPTIONAL. A JSON array of objects representing the structured authorization data associated with the intent, as defined by OAuth 2.0 Rich Authorization Requests {{RFC9396}}. This provides a machine-readable definition of the authorization boundaries (e.g., types of actions, locations, budget limits), complementing the natural language `intent` claim.


In addition to the standard JWT claims, the Intent Token defines two new claims:

*   **intent_id**: RECOMMENDED. A stable and unique identifier for the user's intent. This identifier can be used to correlate multiple token exchange requests that are all part of fulfilling the same high-level intent. Beyond simple auditing, this identifier is critical for enabling stateful security analysis by the Authorization Server, allowing it to correlate multiple requests and detect patterns of behavior over time.

*   **intent**: REQUIRED. A JSON object that represents the user's delegated intent. The structure and semantics of this object are determined by the policies of the issuer and the Authorization Server. It serves as the canonical, auditable record of the user's goal.

### 3.3.1. Example Intent Token Payload

TBD


# 4. Interaction Model using Token Exchange

 TBD


# 5. Security Considerations

## 5.1. Cumulative Intent Drift

A significant risk in agent-based systems is "Cumulative Intent Drift". This occurs when a sequence of individually justifiable actions, when viewed in aggregate, deviates from the user's original high-level intent. An Authorization Server that evaluates each request in isolation may be blind to this drift, as no single request appears malicious on its own.

For example, for an intent of "Help me book a cheap trip to Singapore", an agent might make several requests:
1.  Request `calendar.read` to check schedules (Reasonable).
2.  Request `flight.search` (Core to the intent).
3.  Request `hotel.search` for luxury suites (Diverges from "cheap").
4.  Request `contacts.read` to find colleagues (Potential overreach).

To mitigate this risk, Authorization Servers SHOULD treat the authorization process as stateful. The `intent_id` claim is the essential mechanism for this. The AS SHOULD use the `intent_id` as a correlation key to maintain a history of actions and scopes granted under a single user intent. By performing this sequential analysis, the AS can evaluate new requests in the context of what has already been approved, allowing it to detect anomalous patterns such as privilege escalation or contradiction of the original intent's constraints.

# IANA Considerations

This document has no IANA actions.

# Acknowledgements

The inspiration for the "Intent Token" came from addressing gap 3.1 "authorization context gap" in the Use case and gap analysis. 

The authors would like to thank Steve Tout, Karl McGuinness, and M.Thumilan for commenting, contributing and providing feedback on this draft.

--- back
