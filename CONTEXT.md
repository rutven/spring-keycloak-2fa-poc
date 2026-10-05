# Step-Up Authentication PoC

A standalone demonstration of stronger authentication for sensitive API operations.

## Language

**Standalone PoC**:
An independent demonstration system, separate from the CBS application, used to validate step-up authentication for sensitive API operations.
_Avoid_: CBS feature, production payment approval

**Single-use step-up authorization**:
Authorization established by a fresh OTP verification for at most one human-user invocation of the requested sensitive operation, bound to the login session that completed verification. It is consumed when execution begins, regardless of execution outcome; rejection before execution leaves it unconsumed.
_Avoid_: reusable MFA session, assurance window

**Sensitive operation**:
An action requiring single-use step-up authorization from human users, identified by a stable operation identifier independent of code names or routes. Explicitly trusted integrations are exempt from step-up but still require its business permission.
_Avoid_: sensitive method when referring to the operation's identity

**Authorization handle**:
An opaque reference to a specific single-use step-up authorization, presented by a human caller to identify the authorization intended for consumption. It does not independently establish identity or business permission.
_Avoid_: OTP code, access token

**Demonstration payment**:
A seeded fictional payment with a status of pending, approved, or rejected, used to demonstrate authorization and sensitive operation boundaries. It does not represent funds movement or production payment processing.
_Avoid_: production payment, transfer

**Payment approval**:
The sensitive operation that changes a pending demonstration payment to approved.
_Avoid_: funds transfer, transaction signing

**Payment rejection**:
The sensitive operation that changes a pending demonstration payment to rejected.
_Avoid_: reversal, refund
