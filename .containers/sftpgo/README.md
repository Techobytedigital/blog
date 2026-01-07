# SFTPGo <!-- omit in toc -->

This [SFTPGo container](https://sftpgo.com) mounts the whole repository as an SFTP share, so you can edit files remotely or via a webUI.

> [!WARNING]
> I only use this on my local network, so no special care has been taken to make this a secure deployment. If you're going to expose this to the web, do some research annd make any modifications you need to.
>
> And feel free to contribute those changes back with a pull request! `:)`

## Table of Contents <!-- omit in toc -->

- [SFTPGo Setup](#sftpgo-setup)
  - [SFTPGo WebUI Setup](#sftpgo-webui-setup)

## SFTPGo Setup

- Edit the [`.env` file](./.env.example)
  - Set the absolute path to the [repository root](../../) in `SFTPGO_DATA_DIR`
    - Example: `SFTPGO_DATA_DIR=/home/username/git/techobyte-blog`
    - You can also try with a relative path, like `SFTPGO_DATA_DIR=../../`
- Run `docker compose up -d`

After starting the container, there is some additional [setup to do in the webui](#sftpgo-webui-setup).

### SFTPGo WebUI Setup

- Navigate to your SFTPGo instance.
  - If you did not change the `SFTPGO_HTTP_PORT` in the `.env` file, the address will be `http://<ip-or-hostname>:8080`.
- Follow the setup to create an admin user.
  - Use a strong password.
  - Enable MFA on the account when prompted.
- Create non-admin user for editing.
  - Navigate to `Users` in the webUI.
  - Create a new user, i.e. `sftpuser`, and set your password.
  - Scroll down to `File system`.
  - Set `Storage` to `Local disk`
  - Set `Root directory` to `/data` (this is the path in the container where the local repository is mounted) for the `sftpgo` service's volume mounts.

Now you can log out of SFTPgo and log in as the user you created. Click `Go to WebClient` below the login form. The login header should change from `WebAdmin` to `WebClient`, where you can log in as the `sftpuser` user (or whatever username you used). The "home" directory in `Files` will be the repository's root directory, where you can edit the Hugo site.
