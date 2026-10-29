# Security Policy

## Reporting a vulnerability

If you believe you have found a security issue in Ghost OS, please report it privately.

- **Email:** [support@encryptedlayer.org](mailto:support@encryptedlayer.org?subject=Security%20report) with "Security report" in the subject
- **Include:** the affected page or endpoint, steps to reproduce, and the impact you observed
- **Alternative:** a direct message to [@TryGhostOS](https://x.com/TryGhostOS) on X

Please **do not** open a public GitHub issue, or post details in Discord or on social media, until the issue is fixed.

## Good-faith research

We welcome good-faith security research and will not pursue claims against research that:

- uses only your own accounts and wallets, or test accounts you created;
- does not access, change or move anyone else's data or funds, and stops as soon as the issue is demonstrated;
- does not degrade the service, for example through denial-of-service or high-volume automated testing; and
- keeps details confidential until the issue is resolved.

## Scope

In scope: ghostos.network and the Ghost OS app.

Out of scope: third-party services Ghost OS integrates with (wallets, RPC providers, routers, bridges, data providers), which have their own disclosure programmes; social engineering; physical attacks; and findings that require a compromised user device.

## What Ghost OS guarantees

- Ghost OS never asks for seed phrases.
- Ghost Wallet keys are encrypted at rest, used only to sign actions you approved or your own execution policy allows, and never sent to the browser or to any AI provider.
- In External Wallet mode, your own wallet holds your keys and signs every transaction.
- Every action is shown before signing, and only real transaction hashes are reported.

More: [Security model](https://ghostos.network/docs/security#responsible-disclosure)
