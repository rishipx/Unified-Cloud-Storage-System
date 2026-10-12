# Zero-Knowledge Unified Cloud Storage System with Rclone

This guide shows how to build a unified, cross-device cloud storage system using [Rclone](https://rclone.org). We will combine multiple free Google Drive accounts into a single virtual drive, and create a dedicated, zero-knowledge encrypted drive to secure your sensitive data before it ever leaves your device.

This is a "set-it-and-forget-it" configuration. Using a custom Windows script, drives mount silently at system startup, with no open terminal windows or background apps required. Using [WinFsp](https://winfsp.dev) on Windows and [RSAF](https://github.com/chenxiaolong/RSAF) on Android, the pooled storage mounts directly into your operating system's native file manager. This provides a single directory interface for all connected accounts, bypassing the need to log into each one separately.

<table>
  <tr>
    <td valign="top">

```text
                    Architecture

                    GOOGLE CLOUD
                          │
      ┌─────────┬─────────┴─────────┬─────────┐
      │         │                   │         │
      ↓         ↓                   ↓         ↓
  Account 1  Account 2          Account 3  Account 4
      │         │                   │         │
      └─────────┴─────────┬─────────┴─────────┘
                          │
                          ↓
                     GooglePool:
                          │
                ┌─────────┴─────────┐
                │                   │
                ↓                   ↓
           Normal data        EncryptedData
                │                   │
                ↓                   ↓
                Y:              SecureDrive:
                                    │
                                    ↓
                                    Z:
```

</td>
    <td valign="top"><img src="images/android-file-manager.png" alt="Android file manager showing the mounted drives" width="220"></td>
    <td valign="top"><img src="images/this-pc.png" alt="Windows This PC showing the mounted drives" width="260"></td>
  </tr>
</table>

## What This Setup Achieves

- **Capacity Aggregation:** Combines the 15 GB of free storage from four Google accounts into a single 60 GB drive, managed through one interface instead of manually juggling multiple logins.
- **Zero-Knowledge Privacy:** Encrypts file contents locally before they are uploaded, so Google cannot scan, index, or read your sensitive data.
- **Native OS Integration:** Appears as standard hard drives on Windows and native folders on Android, letting you sync files and stream media directly from the cloud without a web browser.
- **Invisible Background Transfers:** Mounts silently at PC startup and uses a dedicated local cache to process large uploads in the background without freezing your system.

> [!NOTE]
> This architecture is completely modular. You can merge any number of free or paid cloud accounts into a single unified drive. This guide uses four free Google accounts; the same principles apply to other cloud services like Filen. The Filen setup is covered in the [Bonus section](#bonus-filen-architecture-integration) at the end.

---

## Things Needed

**Windows Software**

- [**Rclone**](https://rclone.org/downloads/)
- [**WinFsp**](https://winfsp.dev/rel/)

**Android Software**

- [**RSAF**](https://github.com/chenxiaolong/RSAF)
- **Any SAF-compliant file manager** (e.g., [Material Files](https://github.com/zhanghai/MaterialFiles))

**Cloud & Credentials**

- **Google Accounts** (to form the storage pool)
- [**Google Cloud Custom OAuth Client**](https://rclone.org/drive/#making-your-own-client-id) (Client ID & Secret)

<details>
<summary>Extra info</summary>

**Windows Software**

- **Rclone:** open source tool to sync, manage, and encrypt files across multiple cloud storage providers.
- **WinFsp:** open source Windows driver framework that mounts remote and cloud filesystems as native local drive letters.

**Android Software**

- **RSAF:** mounts Rclone cloud drives directly into your phone's native file manager.

</details>

> [!IMPORTANT]
> Rclone relies on exact name matching to link your accounts together. Whenever you name a drive in this guide (e.g., `drive1`, `GooglePool` or `drive1:`), use that *exact* spelling and capitalization in all later steps, otherwise Rclone will not recognize it.

---

## Process

### Windows

<details>
<summary><b>Step 1:</b> Install <a href="https://rclone.org">Rclone</a> & <a href="https://winfsp.dev">WinFsp</a></summary>

&nbsp;

*The most reliable way to install these tools so Windows automatically recognizes their commands is through PowerShell.*

- **Open PowerShell/Terminal as Administrator and run these commands:**
  - Optional, removes Microsoft telemetry:

    ```powershell
    winget source remove msstore
    ```

  - Downloads both Rclone & WinFsp:

    ```powershell
    winget install Rclone.Rclone WinFsp.WinFsp --accept-source-agreements
    ```

- **Crucial:** Restart PowerShell and run `rclone version`. If the version number appears, you are ready to proceed.

*(If you prefer, you can also download and install these manually from their official websites.)*

</details>

<details>
<summary><b>Step 2:</b> Generate your free <b>Google Cloud Custom OAuth Client ID and Secret</b></summary>

&nbsp;

1. Log in to the [Google Cloud Console](https://console.cloud.google.com) with any of your Google accounts.
2. Click **Select a project** at the top left & click the **NEW PROJECT** button in the popup.

   <img src="images/select-project.png" alt="Select a project button" width="49%"> <img src="images/new-project.png" alt="New project button" width="49%">

3. Name the project whatever you want (e.g., `rclone`) and leave the **Organization** setting as `No organization`.

   ![Name the project](images/name-project.png)

4. Go to the home page, click **Select a project** again, then select your newly created project and open it.

   ![Select your project](images/project-list.png)

5. Search for "Google Drive API" in the search bar, then click it and **Enable** it.

   <img src="images/search-drive-api.png" alt="Search for Google Drive API" width="49%"> <img src="images/enable-api.png" alt="Enable the API" width="49%">

6. This opens the **APIs & Services** page. If not, go to the home page and select **APIs & Services**.
7. On the left sidebar click **OAuth consent screen** & select **Get Started**.

   <img src="images/oauth-consent-screen.png" alt="OAuth consent screen" width="49%"> <img src="images/oauth-get-started.png" alt="Get started" width="49%">

8. On this page enter the following details:
   1. Type in the **App Name** (e.g., `rclone`) & an **Email**.

      ![App information](images/app-info.png)

   2. Select Audience type **External**.

      ![Audience type](images/audience-external.png)

   3. Enter an email address under contact information.
   4. Finish.
   5. Click **Create**.

9. Under the **Branding** tab just fill in `https://rclone.org` in the respective fields & `rclone.org` under **Authorised domains**, then click **Save**.

   ![Branding tab](images/branding.png)

   ![Authorised domains](images/branding-cropped.png)

10. Under the **Audience** tab click **Publish app** & confirm.

    ![Publish app](images/publish-app.png)

11. In the **Clients** tab click **Create client**, set Application type to **Desktop app**, give it a name and click **Create**.

    <img src="images/create-client.png" alt="Create client" width="49%"> <img src="images/client-desktop-app.png" alt="Desktop app client" width="49%">

12. Your Client ID & Client Secret will be presented. Save both in a secure place, you need them in the next step.

    ![Client ID and secret](images/client-id-secret.png)

</details>

<details>
<summary><b>Step 3:</b> Connect Your Google Accounts to Rclone</summary>

&nbsp;

- Open Command Prompt or PowerShell in Windows.
- Type `rclone config` and press Enter.
- Type `n` to create a new remote and name it, e.g. `drive1`.

  ```text
  PS C:\WINDOWS\system32> rclone config
  No remotes found - make a new one

  e) Edit existing remote
  n) New remote
  d) Delete remote
  r) Rename remote
  c) Copy remote
  s) Set configuration password
  q) Quit config
  e/n/d/r/c/s/q> n
  ```

- Now type `drive` or `24` to select **Google Drive** from the list of storage options.
- Paste the **Client ID** and **Client Secret** you generated in Step 2.
- Choose scope `1` (Full access all files).

  ![Paste client ID and secret](images/paste-client-id.png)

- When you reach the `service_account_file>` prompt, just press **Enter** to leave it blank.
- **Skip advanced config:** when prompted `Edit advanced config?`, type `n` and press **Enter**.

  ![Skip advanced config](images/skip-advanced-marked.png)

- Type `y` for web browser authentication. Your browser will open; log in with your **first** Google account and grant permissions.

  ![Choose the first Google account you want to log in with](images/choose-account.png)

  *Choose the first Google account you want to log in with.*

  ![Click Show advanced, then Go to rclone (unsafe)](images/unverified-app.png)

  *Click **Show advanced**, then **Go to rclone (unsafe)**.*

  ![Click Continue to connect](images/allow-access.png)

  *Click **Continue** to connect.*

  Now go back to the PowerShell window.

- **`Configure this as a Shared Drive (Team Drive)?`**

  ![Type n and press Enter](images/shared-drive-no.png)

  *Type `n` & press Enter.*

- The new remote's details will be displayed in the terminal for verification. If everything looks good, type **`y`** & press Enter.

  ![Confirm the remote](images/confirm-remote.png)

- Your `drive1` is added.

  ```text
  Current remotes:

  Name                 Type
  ====                 ====
  drive1               drive

  e) Edit existing remote
  n) New remote
  d) Delete remote
  r) Rename remote
  c) Copy remote
  s) Set configuration password
  q) Quit config
  e/n/d/r/c/s/q>
  ```

- Repeat this entire process for `drive2`, `drive3`, and `drive4`, logging into the respective Google account each time.

</details>

<details>
<summary><b>Step 4:</b> Create the Unified Storage Pool</summary>

&nbsp;

- Type `rclone config` in your terminal.
- Type `n` for a new remote and name it, e.g. `GooglePool`.
- Now type **`union`** or **`62`** to select union from the list of storage types.
- When asked for upstreams, enter the names of the accounts you just created, separated by spaces: `drive1: drive2: drive3: drive4:`

  ![Union upstreams](images/union-upstreams.png)

- For the next 5 options (action policy, create policy, search policy, cache time, advanced config) just press Enter to accept the defaults, which are perfectly fine for this setup.
- When the configuration is complete, type `y` and press Enter to save the remote.

  ![Save the union remote](images/union-save.png)

- Your remotes list will now look like this:

  ```text
  Current remotes:

  Name                 Type
  ====                 ====
  GooglePool           union
  drive1               drive
  drive2               drive
  drive3               drive
  drive4               drive

  e) Edit existing remote
  n) New remote
  d) Delete remote
  r) Rename remote
  c) Copy remote
  s) Set configuration password
  q) Quit config
  e/n/d/r/c/s/q>
  ```

</details>

<details>
<summary><b>Step 5:</b> Create the Encrypted Drive (SecureDrive)</summary>

&nbsp;

- Type `rclone config` again.
- Type `n` for a new remote and name it, e.g. `SecureDrive`.
- Type **`crypt`** or **`16`** to select crypt from the list of storage types.
- For the remote to encrypt, point it to a specific folder inside your pool, e.g. `GooglePool:EncryptedData`

  ![Crypt remote path](images/crypt-remote.png)

- Select an encryption option: choose `1` (standard) for filename encryption, and `1` (true) for directory name encryption.

  ![Filename and directory name encryption](images/crypt-directory-names.png)

- **Password:** Generate or enter a strong password.

  > [!CAUTION]
  > Save this in a password manager. If you lose it, your encrypted data is gone permanently (unencrypted data stays).

- **Salt:** Enter a strong salt password (optional but highly recommended for extra security).
- When asked `Edit advanced config?`, type `n` & press Enter.
- Finish and save the remote, then quit the config menu (`q`).
- The final structure will look like this:

  ```text
  Current remotes:

  Name                 Type
  ====                 ====
  GooglePool           union
  SecureDrive          crypt
  drive1               drive
  drive2               drive
  drive3               drive
  drive4               drive

  e) Edit existing remote
  n) New remote
  d) Delete remote
  r) Rename remote
  c) Copy remote
  s) Set configuration password
  q) Quit config
  e/n/d/r/c/s/q>
  ```

</details>

<details>
<summary><b>Step 6:</b> Auto-Mount the Drives in Windows via Startup Script (WinFsp)</summary>

&nbsp;

We will use a Windows background script to mount the cloud drives automatically and silently, so you never have to leave a terminal window open.

- Press **Windows Key + R**, type `shell:startup`, and press Enter. This opens your Windows Startup folder.
- Right-click inside the folder, select **New > Text Document**, and open it.
- Paste the following code into the document:

  ```vbscript
  Set WshShell = CreateObject("WScript.Shell")

  ' Mount the Encrypted Z: Drive
  WshShell.Run "rclone mount SecureDrive: Z: --vfs-cache-mode full --cache-dir D:\RcloneCache --links", 0, False

  ' Mount the Unencrypted Combined Y: Drive
  WshShell.Run "rclone mount GooglePool: Y: --vfs-cache-mode full --cache-dir D:\RcloneCache", 0, False
  ```

- Save the file and close Notepad.
- Rename the text file to **`RcloneStartup.vbs`** (make sure you delete the `.txt` extension when renaming).

  ![RcloneStartup.vbs in the Startup folder](images/startup-folder.png)

- **Double-click the new `.vbs` file.** Your `Z:` and `Y:` drives will instantly appear in "This PC", and they will mount automatically in the background every time you turn on your computer.

  ![Mounted drives in This PC](images/this-pc.png)

> [!TIP]
> By default, when you drop a file into the pool, Rclone saves it to the account with the most free storage. To force a file onto a specific account, log in to your Google Drive accounts via the web or the Google Drive Android app and create a uniquely named folder in each one (e.g., `Drive 1`, `Drive 2`, `Drive 3`). Now, whenever you drop a file into the `Drive 1` folder in your pool, it will strictly upload to that specific account.

</details>

---

## Process

### Android

To access this exact same setup on your phone natively without syncing locally:

- **Step 1: Export your Windows Rclone config**
  - Open a terminal and type `rclone config file`.
  - Open the path provided (usually `C:\Users\YourName\AppData\Roaming\rclone\rclone.conf`).
  - Copy & export this `rclone.conf` file to your Android phone's local storage.

  > [!WARNING]
  > `rclone.conf` holds your OAuth tokens and obscured (not encrypted) crypt passwords. Transfer it over a trusted channel and delete the copy from the phone's storage after importing.

- **Step 2: Configure RSAF**
  - Install **RSAF** (available via [GitHub](https://github.com/chenxiaolong/RSAF) or [F-Droid](https://f-droid.org/packages/com.chiller3.rsaf/)).
  - Open RSAF, tap **Import configuration** and select the `rclone.conf` file you copied from your PC.

    <img src="images/rsaf-import.png" alt="RSAF import configuration" width="300">

  - The app will instantly recognize your `GooglePool` and `SecureDrive` setups, including your encryption keys.

<details>
<summary><b>Step 3:</b> Access via File Manager</summary>

&nbsp;

- Open a SAF-compliant app like **Material Files**.
- Click the hamburger icon and tap **Add storage**.

  <img src="images/material-add-storage.jpg" alt="Material Files: Add storage" width="300">

- Click **External storage**, which opens the native Android Storage Access Framework picker.

  <img src="images/material-external-storage.jpg" alt="Material Files: External storage" width="300">

- Click the hamburger icon again, scroll down, select your **RSAF** drives from the list (`SecureDrive` & `GooglePool`) and allow access.

  <img src="images/material-rsaf-drives.jpg" alt="Selecting the RSAF drives" width="300">

- Your cloud drive is now mounted as a native directory on your phone. You can stream videos or open documents directly without downloading them first.

  <img src="images/android-file-manager.png" alt="Cloud drives mounted in Material Files" width="300">

</details>

---

## Bonus: Filen Architecture Integration

<details>
<summary>Show steps</summary>

&nbsp;

You can integrate a zero-knowledge provider like **Filen** alongside your Google pool. Because Filen handles its own client-side encryption and does not use standard OAuth like Google, the setup requires generating a raw API key using their official Command Line Interface (CLI) tool.

- **Step 1: Generate your API key**
  - Download the `x64` (or `amd64`) release of the Filen CLI from their official GitHub repository.
  - Extract the ZIP, open the extracted folder, click the File Explorer address bar, type `powershell`, and press Enter.
  - Log in to your account by running `.\filen-cli.exe login` and entering your Filen email and password.
  - Run the command `export-api-key` and copy the long string it prints. Save this securely.
- **Step 2: Add Filen to Windows Rclone**
  - Open a new normal PowerShell window and run `rclone config`.
  - Press `n` for a new remote and name it `FilenDrive`.
  - Choose `filen` as the storage type.
  - When prompted, enter your Filen email, password, and the unique API key you just generated.
- **Step 3: Add to your VBS startup script**
  - Press **Windows Key + R**, type `shell:startup`, and press Enter.
  - Right-click your existing `RcloneStartup.vbs` script, click **Edit**, and paste these two lines at the very bottom to mount your new Filen drive silently as `X:`:

    ```vbscript
    ' Mount the Filen Drive as X:
    WshShell.Run "rclone mount FilenDrive: X: --vfs-cache-mode full --cache-dir D:\RcloneCache", 0, False
    ```

  - Save and close Notepad.
- **Step 4: Sync the update to Android**
  - Because you added this new account to your PC, you need to update your phone so it can see it too.
  - Press **Windows Key + R**, type `%appdata%\rclone`, and press Enter.
  - Send the updated `rclone.conf` file to your phone's `Downloads` folder.
  - Open the **RSAF** app on your phone, tap **Import configuration**, and select the new `rclone.conf` file. Your Filen drive will now appear natively alongside your Google drives.

</details>

---

## Resources

- [Rclone backend support tier list](https://rclone.org/tiers/): shows supported cloud storage options.
- [Rclone Official Documentation](https://rclone.org/docs/)
