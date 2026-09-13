# Azure AD B2C passwordless email sign-in

Custom policies for a passwordless sign-up and sign-in journey in Azure AD B2C, using email verification instead of a password.

Built on the [Azure AD B2C custom policy starter pack](https://github.com/Azure-Samples/active-directory-b2c-custom-policy-starterpack) and Microsoft's passwordless sign-in sample. `TrustFrameworkBase.xml` and `TrustFrameworkExtensions.xml` are the starter pack files; the passwordless journey is in the two files that build on top of them.

## Files

| File | Role |
| --- | --- |
| `TrustFrameworkBase.xml` | Starter pack base. Claims schema, claims providers, base user journeys. Not modified. |
| `TrustFrameworkExtensions.xml` | Starter pack extensions. Identity Experience Framework app registration and social IdP wiring. |
| `TrustFrameworkExtensions_passwordless_only.xml` | The passwordless journey: custom technical profiles and the `SignUpOrSignInPasswordless` user journey. |
| `SignUpOrSignin_passwordless.xml` | Relying party policy. The endpoint an application actually points at. |

## How the journey works

Azure AD B2C requires every local account to carry a password in the directory, so a passwordless journey cannot simply omit one. This implementation works around that:

**Sign up.** `LocalAccountSignUpWithLogonEmail-Custom` drops the `newPassword` and `reenterPassword` claims from the sign-up form, so the user is never asked for a password. Email ownership is proven by the `Verified.Email` claim, which triggers B2C's built-in one-time-code email verification. `AAD-UserWriteUsingLogonEmail-Custom` then runs the `CreateRandomPassword` transformation to generate a GUID and persists it as the account password. The user never sees it and it is never used for authentication.

**Sign in.** `LocalAccountDiscoveryUsingEmailAddress-SignIn` reuses the password-reset technical profile, which verifies the email address by one-time code and returns the object ID. Because sign-in runs through email verification rather than credential validation, the stored random password is never presented.

The account is created with `DisablePasswordExpiration, DisableStrongPassword`, since an unused GUID password should not expire and does not need to satisfy complexity rules.

**Refresh tokens.** The relying party exposes a `Token` endpoint bound to the `RedeemRefreshToken` journey, which runs `AssertRefreshTokenIssuedLaterThanValidFromDate` against the directory. Revoking a user's refresh tokens in Azure AD therefore invalidates silent reauthentication rather than leaving an issued refresh token usable until expiry.

## Security notes and trade-offs

- **Email possession is the only factor.** Anyone who controls the mailbox controls the account. That is the point of the design, but it means the mailbox provider's own security becomes your authentication boundary. For higher-assurance scenarios, add a second factor rather than relying on this alone.
- **The random password is a real credential in the directory.** It is unused and unknown to the user, but it exists. Password reset and any other journey that accepts a password should be excluded from a tenant running this policy, or an attacker who resets it gains a conventional login path.
- **Account enumeration.** The sign-in technical profile returns a distinct error when no account exists for the supplied address, which reveals whether an email is registered. Worth customising the error message where enumeration matters.
- **One-time code delivery is unauthenticated input.** Rate limiting and lockout behaviour should be verified against your threat model rather than assumed from the sample.

## Deploying

Replace `yourtenant.onmicrosoft.com` throughout with your own tenant, register the `IdentityExperienceFramework` and `ProxyIdentityExperienceFramework` applications, substitute their application IDs in `TrustFrameworkExtensions.xml`, and create the `B2C_1A_TokenSigningKeyContainer` and `B2C_1A_TokenEncryptionKeyContainer` policy keys. Upload in order: base, extensions, passwordless extensions, then the relying party policy.

No tenant identifiers, application IDs, or keys are committed to this repository.

## License

MIT. Starter pack content remains under its original Microsoft license.
