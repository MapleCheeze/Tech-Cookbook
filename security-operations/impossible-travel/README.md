# Impossible Travel Detection & Response

End-to-end SOAR workflow for impossible-travel sign-ins in Microsoft Sentinel. Detection fires the analytic rule, enrichment runs automatically, the workflow contacts the user, and conditional access actions are invoked or escalation happens based on the user's response and the surrounding signal, so analysts only see the cases worth their time.

Companion writeup: [portfolio site](https://maplecheeze.github.io/Portfolio/portfolio/impossible-travel-workflow/)

## Context

Out-of-the-box impossible-travel alerts are some of the noisiest signals in Sentinel. VPN traffic, proxy egress, cloud-hosted clients, and legitimate travel all produce signal that looks like compromise. Triaging these manually burns analyst hours on false positives, and the genuine cases get lost in the noise.

This workflow handles the routine 90% automatically, enriches the alert, contacts the user, evaluates the response, and applies conditional access when warranted, and only escalates to an analyst when signals genuinely suggest compromise.

## Architecture

The Logic App is triggered by an impossible-travel analytic rule in Sentinel and runs through:

1. **Enrichment**: pulls user risk score from Entra ID Protection, recent sign-in patterns, device compliance state, MFA history on the session, and ASN reputation for the source IPs.
2. **User verification**: posts a confirmation prompt to the user (Teams or email) asking them to confirm or deny the sign-in.
3. **Conditional response** --
   - User confirms and signals are clean → close as benign.
   - User denies, doesn't respond, or signals are suspicious → invoke a conditional access policy to revoke sessions, require MFA, or block.
   - Signals strongly suggest compromise regardless of user response → escalate to an analyst with full enrichment context.
4. **Notification & logging**: analyst escalations post to the SOC channel with the enrichment summary, and the action trail is written back to the Sentinel incident.

The boundary between auto-handle and escalate is the highest-value design decision. Too generous and the analyst queue floods; too narrow and the workflow misses real incidents.

## Components

- [`ImpossibleTravelLogicApp.json`](ImpossibleTravelLogicApp.json), Logic App workflow. Tenant, subscription, RG, Sentinel workspace, and connector references are placeholders.
- [`ImpossibleTravelV3.sanitized.svg`](ImpossibleTravelV3.sanitized.svg), workflow diagram.

## Deployment

You will need:

- Microsoft Sentinel with the impossible-travel analytic rule enabled
- Entra ID Protection (for user risk score) and Conditional Access (for response actions)
- Defender for Cloud Apps or equivalent for ASN reputation and session controls
- A messaging connector (Teams or Outlook) for user prompts
- An Azure subscription for the Logic App

Steps:

1. Open `ImpossibleTravelLogicApp.json` and replace placeholders for your environment.
2. Configure the analytic rule in Sentinel to trigger this Logic App as an automated response.
3. Wire up the conditional access action, ideally a named policy with the specific control you want this workflow to invoke (session revoke, MFA challenge, or block).
4. Configure the messaging connector for user prompts. Test the prompt UX before going live.
5. Define your VPN, proxy, and infrastructure ASN carve-outs at the analytic rule level so they don't fire alerts in the first place.
6. Test on a synthetic event before exposing to live traffic.

## Considerations

- **Carve-outs belong on the rule, not the Logic App**: corporate VPN exits, cloud-hosted clients, and known infrastructure ASNs should be suppressed at the analytic rule. Filtering inside the Logic App means you're still firing alerts you ignore.
- **Executive travel patterns**: some users travel constantly. Either tier the policy (looser for known travelers) or accept those users won't benefit from this control.
- **User-prompt fatigue**: if the system prompts a user too often, they'll click through without reading. Tune the rule to prompt only on signals worth the user's attention.
- **Session revocation has user impact**: revoking sessions logs the user out of every active session. Match the response action to the signal severity, don't over-rotate on weak signal.
- **Escalation criteria need iteration**: the auto-handle vs escalate boundary will not be right on day one. Plan for tuning runs, ideally with the SOC owning the criteria.
