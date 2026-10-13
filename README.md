This repository contains a manual workflow that starts a GitHub-hosted Windows runner (`windows-latest` by default, or `windows-2022` when the **Use the Windows Server 2022 runner image** option is enabled), connects it to your private Tailscale network, enables a temporary RDP account with local administrator privileges, and keeps the session available for a limited time. When the job finishes or is canceled, RDP is disabled and the temporary account is removed. The official Tailscale action also disconnects and removes the ephemeral node created for the workflow [1].

> GitHub-hosted runners are ephemeral. Do not store important files on them or use this workflow as a permanent computer.

## Prepare Tailscale

Create an account on [Tailscale](https://login.tailscale.com/start) and open the [Keys](https://console.tailscale.com/admin/settings/keys) page in the administration console. Select **Generate auth key** and configure the key with an identifiable description, such as `github-windows-rdp`.

For this use case, enable **Reusable**, **Ephemeral**, and **Tags**. If your tailnet uses device approval, also enable **Pre-approved**. Assign an exclusive tag, such as `tag:github-rdp`. Tailscale documentation recommends that keys used by GitHub Actions have a tag identity and be reusable and ephemeral [1] [2].

If Tailscale requires the tag to exist before generating the key, first create `tag:github-rdp` in the access controls configuration. Your tailnet access policy must allow your personal device to connect to TCP port `3389` on nodes with that tag.

After selecting **Generate key**, copy the key only once. Do not publish it or paste it into this chat.

## Repository secrets

In GitHub, open **Settings → Secrets and variables → Actions → New repository secret** and create these three secrets:

| Name | Content | Purpose |
|---|---|---|
| `TAILSCALE_AUTHKEY` | The Tailscale auth key | Temporarily connect the runner to the tailnet |
| `RDP_USERNAME` | A temporary user, such as `rdpuser` | Local account created for RDP |
| `RDP_PASSWORD` | At least 8 characters, with at least 3 of these categories: uppercase letters, lowercase letters, numbers, and symbols; it must not contain `RDP_USERNAME` | Password for the temporary account |

Never write these values directly in the YAML, README, an issue, or a commit. The auth key is a private credential; Tailscale recommends protecting it and using ephemeral keys for temporary nodes [2].

## Prepare the client device

Install the [Tailscale](https://tailscale.com/download) application on the device you will use to connect, sign in to the same tailnet, and verify that it can see the authorized devices. On Android, you can use a compatible Remote Desktop client together with the Tailscale application; on Windows, you can use **Microsoft Remote Desktop**.

## Run the session

Open **Actions**, select **Windows RDP via Tailscale**, click **Run workflow**, and choose a duration between 15 and 300 minutes. Leave **Use the Windows Server 2022 runner image** unchecked to keep the default `windows-latest` runner, or check it to select the `windows-2022` image. For the first test, select `15` or `30` minutes.

When the **Keep RDP session available** step starts, its logs will show a `100.x.x.x` address. This is the runner's private Tailscale IPv4 address. On a client connected to the same tailnet, open Remote Desktop and connect to that address using port `3389`, for example:

```text
100.101.102.103:3389
```

Use the value stored in `RDP_USERNAME` as the username and the value stored in `RDP_PASSWORD` as the password. The account is temporarily added to `Administrators`, so a new RDP session should be able to approve UAC prompts without requesting the `runneradmin` password. The password must comply with the Windows complexity policy: at least 8 characters, at least 3 categories among uppercase letters, lowercase letters, numbers, and symbols, and it must not include the username. Twelve or more characters are recommended. Do not use the previous Cloudflare hostname: this version no longer needs Cloudflare or any domain.

Cancel the workflow when you are finished. If you do not cancel it, it will stop automatically when the selected duration is reached. Cleanup disables RDP and removes the temporary account; Tailscale removes the ephemeral node after the action finishes [1].

## Troubleshooting

If the workflow fails at **Validate inputs and secrets**, check that `TAILSCALE_AUTHKEY`, `RDP_USERNAME`, and `RDP_PASSWORD` exist exactly as named. If the Tailscale action fails, verify that the key has not expired, has an allowed tag, and, when applicable, is pre-approved.

If RDP does not connect, confirm that your client device is connected to the same tailnet, that the access policy allows traffic to `tag:github-rdp` on port `3389`, and that the job has reached the **Keep RDP session available** step. If you switched to this version while already connected, close the RDP session and run the workflow again: Administrators permissions are applied to a new session. The private address is valid only while the workflow remains active.

## References

[1]: https://tailscale.com/docs/integrations/github/github-action "Tailscale GitHub Action"
[2]: https://tailscale.com/docs/features/access-control/auth-keys "Tailscale Auth keys"
