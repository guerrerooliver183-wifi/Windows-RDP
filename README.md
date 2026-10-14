# Windows-RDP Temporary Session

This repository starts a temporary Windows runner, connects it to your Tailscale network, and enables Remote Desktop (RDP) for the selected session duration. Connect only from a device that is also signed in to the same Tailscale tailnet.

The runner is temporary. The workflow disables RDP and removes the temporary account during cleanup when the job ends or is canceled.

## Requirements

- A GitHub account with permission to add repository Actions secrets and run workflows. GitHub requires write access to manually run a `workflow_dispatch` workflow. [5] [6]
- A Tailscale account and tailnet. If you do not have one yet, create it with a supported identity provider and add your client device(s) to the tailnet. [1]
- The Tailscale client installed and connected on the device you will use for RDP. [3]
- Three GitHub Actions repository secrets, described below.

## 1. Create or prepare your Tailscale tailnet

If you already use Tailscale, use your existing account and tailnet. Otherwise:

1. Go to [Tailscale Get Started](https://tailscale.com/) and sign up with a supported identity provider.
2. Choose personal or business use and complete the initial setup.
3. Install Tailscale on the device from which you will connect:
   - [Windows](https://tailscale.com/download/windows): install the client and sign in with the same identity provider and tailnet.
   - [Android](https://play.google.com/store/apps/details?id=com.tailscale.ipn): install Tailscale, choose **Get Started**, approve the Android VPN configuration, and sign in to the same tailnet. [3]

Keep Tailscale connected on the client before starting the RDP connection. The Windows runner joins the tailnet automatically when the workflow runs.

### Create a Tailscale auth key for the runner

The workflow needs an auth key because its GitHub-hosted runner must join your tailnet without an interactive browser sign-in. An auth key is a credential: never paste it into the workflow file, a commit, an issue, or a chat message. [2]

1. Open the [Tailscale Keys page](https://login.tailscale.com/admin/settings/keys) in the admin console.
2. Select **Generate auth key**.
3. Give it a recognizable description, such as `GitHub Actions Windows-RDP`.
4. Enable **Reusable**, because every workflow run starts a new runner and needs to authenticate that new device.
5. Enable **Ephemeral**. Tailscale automatically removes an ephemeral device after a short period of inactivity, normally 30–60 minutes after it goes offline. [10]
6. Set an expiration date. Tailscale auth keys can be configured to expire between 1 and 90 days; create a replacement and update the GitHub secret before it expires. [2]
7. If device approval is enabled for your tailnet, enable **Pre-approved** or approve the runner from the Tailscale admin console when it appears. Do not add tags unless you have intentionally configured your tailnet policy for them.
8. Generate the key and copy it directly into the GitHub secret in the next section. Do not expose it in logs or source code.

Reusable keys can authenticate multiple devices, so protect this key carefully. If it is exposed, revoke it in the Tailscale admin console and replace the GitHub secret. Revoking an auth key does not itself remove devices already registered with that key; remove an unwanted device from the Tailscale Machines page. [2]

If your tailnet uses custom ACLs or grants, make sure the client device is allowed to reach the temporary runner on TCP port `3389`. Tailscale access rules can limit traffic by destination, protocol, and port. [4]

## 2. Add the GitHub repository secrets

In the GitHub repository, open **Settings → Secrets and variables → Actions → Secrets → New repository secret**. Add these three secrets: [5]

- `RDP_USERNAME` — the local Windows username to create for the RDP session. It must be 1–20 characters and contain only letters, numbers, periods (`.`), hyphens (`-`), or underscores (`_`). This account is added to the Windows **Administrators** group so it can elevate without a UAC credential prompt during the session.
- `RDP_PASSWORD` — the password for `RDP_USERNAME`. It must be at least 8 characters, must not contain the username, and must include at least 3 of these 4 categories: uppercase letters, lowercase letters, numbers, and symbols.
- `TAILSCALE_AUTHKEY` — the reusable Tailscale auth key generated in the previous section.

Use a strong, unique `RDP_PASSWORD`. The account is an administrator on the temporary runner, and the workflow configures administrator elevation to proceed without a UAC prompt for that session. That means actions requiring elevation will not stop for a confirmation prompt. Never reuse this password for another account.

GitHub masks secret values in workflow logs, but do not deliberately print them. If a required secret is missing or invalid, the validation step stops the job before RDP is enabled.

## 3. Run the workflow

The default branch is `master`, and the workflow must be present on the default branch for GitHub's manual **Run workflow** control to be available. [6]

1. Open the repository on GitHub and select **Actions**.
2. Choose **Start temporary Windows RDP session** in the workflow list.
3. Select **Run workflow** and choose the `master` branch.
4. Set the available inputs:
   - **Use the Windows Server 2022 runner image** — off by default. When off, the workflow uses `windows-latest`; when on, it uses `windows-2022`.
   - **Maximum duration of the temporary session** — choose 15, 30, 60, 120, 180, 240, or 300 minutes. The default is 60 minutes.
   - **Install Mesa3D software OpenGL driver system-wide** — on by default. Mesa3D provides CPU-based software OpenGL rendering; it is not GPU acceleration.
5. Select **Run workflow** again to start the job.
6. Open the running job's logs. Wait for the **Keep RDP session available** step. The workflow prints the runner's Tailscale IPv4 address and the RDP connection address in the form `100.x.x.x:3389`.

The session ends when the selected duration elapses, the workflow is canceled, or the job otherwise stops. The cleanup step attempts to disable RDP, restore the runner's previous UAC setting, and remove the temporary user and its group memberships.

## 4. Connect with RDP

First confirm that Tailscale is connected on the client device and that it is logged in to the same tailnet as the runner.

### Windows

1. Open **Remote Desktop Connection** (it is included with Windows). You can press **Start**, type `Remote Desktop Connection`, and open the app. [9]
2. In **Computer**, enter the address printed by the workflow, including port `3389`, for example `100.88.12.34:3389`.
3. Select **Connect**.
4. Enter the username configured in `RDP_USERNAME` and the corresponding `RDP_PASSWORD`.

### Android

1. Install [Tailscale for Android](https://play.google.com/store/apps/details?id=com.tailscale.ipn) and sign in to the same tailnet as the runner.
2. Install [Windows App from Google Play](https://play.google.com/store/apps/details?id=com.microsoft.rdc.androidx). Microsoft's listing currently specifies Android 11 or later. [8]
3. Open Windows App and add a remote PC. Select the **+** or **Add** option, then enter the workflow's RDP address, including `:3389`, as the PC name. Microsoft's Windows App guide describes adding the remote PC by its IP address and then entering the account for that PC. [7]
4. Select **Add & Connect** (or save the PC and tap its card).
5. Enter the username configured in `RDP_USERNAME` and its `RDP_PASSWORD`. Accept the certificate prompt if shown and if the address matches the current workflow run.

The Tailscale IP belongs to that temporary runner. Use the current run's address; it will not be the same for a future run.

## Security notes

- The workflow does **not** require public port forwarding. Do not expose TCP port `3389` to the public internet. RDP access should travel through the Tailscale tailnet only.
- Treat `TAILSCALE_AUTHKEY` and `RDP_PASSWORD` as credentials. Keep them in GitHub Actions secrets and rotate them if they are exposed.
- The temporary RDP account is an administrator, and administrator UAC prompts are suppressed during the session to enable silent elevation. Only run this workflow from a repository and workflow definition you trust, and restrict who can write to the repository and access the tailnet.
- If a Tailscale auth key expires, create a new one and update `TAILSCALE_AUTHKEY` in GitHub. If access controls are restrictive, verify that the client may reach the runner on TCP `3389`.

## References

[1]: https://tailscale.com/docs/how-to/quickstart "Tailscale quickstart"
[2]: https://tailscale.com/docs/features/access-control/auth-keys "Tailscale auth keys"
[3]: https://tailscale.com/docs/install/android "Install Tailscale on Android"
[4]: https://tailscale.com/docs/features/access-control/grants "Tailscale grants and network permissions"
[5]: https://docs.github.com/actions/security-guides/using-secrets-in-github-actions "Using secrets in GitHub Actions"
[6]: https://docs.github.com/actions/managing-workflow-runs/manually-running-a-workflow "Manually running a workflow"
[7]: https://learn.microsoft.com/en-us/windows-app/get-started-connect-devices-desktops-apps?tabs=windows-avd%2Cwindows-w365%2Cwindows-devbox%2Candroid-rds%2Candroid-pc "Get started with Windows App to connect to devices and apps"
[8]: https://play.google.com/store/apps/details?id=com.microsoft.rdc.androidx "Windows App on Google Play"
[9]: https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remotepc/remote-desktop-allow-access "Enable Remote Desktop and connect to a PC"
[10]: https://tailscale.com/docs/features/ephemeral-nodes "Tailscale ephemeral nodes"
