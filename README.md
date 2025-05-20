# MUMBLE VOIP IN QUBES OS

Instructions for configuring [Mumble↗](https://en.wikipedia.org/wiki/Mumble_%28software%29) server and client with Tor-routing in [Qubes OS↗](https://qubes-os.org). Both server and client are securely ran in separate _disposable_ [Whonix↗](https://whonix.org) Workstations.

> [!IMPORTANT]
> The Whonix version number _(currently 17)_ is liable to change. If a more recent version is available, e.g., `whonix-workstation-18-dvm`, use that instead by updating commands which use the outdated reference.

## Requirements
- 8Gb Memory _(16Gb or more recommended)_
- Qubes OS 4.1 or higher w/Whonix VMs installed
- A moderate understanding of Qubes OS

## Table of Contents
- [Mumble Server Setup](#mumble-server-setup)
    - [1. Create & Configure Mumble Server TemplateVM](#1-create--configure-mumble-server-templatevm)
    - [2. Create & Configure Mumble Server AppVM & DispVM](#2-create--configure-mumble-server-appvm--dispvm)
    - [3. Configure Tor Hidden Service For Mumble Server](#3-configure-tor-hidden-service-for-mumble-server)
- [Mumble Client Setup](#mumble-client-setup)
    - [1. Create & Configure Mumble Client TemplateVM](#1-create--configure-mumble-client-templatevm)
    - [2. Create & Configure Mumble Client DispVM](#2-create--configure-mumble-client-dispvm)
    - [Optimal Mumble Client Settings](#optimal-mumble-client-settings)

<br>

# Mumble Server Setup

## 1. Create & Configure Mumble Server TemplateVM

1.  [**user**@**dom0**]() Create Mumble server _TemplateVM (`murmurd-ws-17`)_

    1. Clone `whonix-workstation-17`
    
        ```bash
        qvm-clone whonix-workstation-17 murmurd-ws-17
        ```

    2. Update new template preferences _(recommended)_

        1. `qvm-prefs murmurd-ws-17 maxmem 2000`
        2. `qvm-prefs murmurd-ws-17 memory 200`

2.  [**user**@**dom0**]() Update `murmurd-ws-17`

    ```bash
    qubes-vm-update --force-update --targets murmurd-ws-17
    ```

3.  [**user**@**dom0**]() Launch `murmurd-ws-17`

    ```bash
    setsid qvm-run murmurd-ws-17 xfce4-terminal &>/dev/null
    ```

4.  [**user**@**murmurd-ws-17**]() Create service directory for `mumble-server`

    ```bash
    sudo mkdir -p /etc/systemd/system/mumble-server.service.d
    ```

5. [**user**@**murmurd-ws-17**]() Prevent `mumble-server.service` from running in _persistent_ VMs

    ```bash
    cat << 'EOF' | sudo tee -a /etc/systemd/system/mumble-server.service.d/override.conf &>/dev/null
    [Unit]
    ConditionPathExists=/run/qubes/persistent-none
    EOF
    ```

6.  [**user**@**murmurd-ws-17**]() Install `mumble-server` package

    ```bash
    sudo apt install -y mumble-server
    ```

7.  [**user**@**murmurd-ws-17**]() Shutdown the template

    ```bash
    sudo poweroff
    ```

<br>

## 2. Create & Configure Mumble Server AppVM & DispVM

1.  [**user**@**dom0**]() Create Mumble Server _AppVM_ _(`murmurd-dvm`)_

    <details>
    <summary><b>Step Details</b></summary>

    > This VM is setup as a _DispVM template_ for the named _DispVM_ that'll be running the server.

    </details>

    ```bash
    qvm-create murmurd-dvm -t murmurd-ws-17 -l black --prop netvm='' --prop template_for_dispvms=True --prop default_dispvm=''
    ```

2.  [**user**@**dom0**]() Enable `appmenus-dispvm` feature in `murmurd-dvm`

    ```bash
    qvm-features murmurd-dvm appmenus-dispvm 1
    ```

3. [**user**@**dom0**]() Launch `murmurd-dvm`

    ```bash
    setsid qvm-run murmurd-dvm xfce4-terminal &>/dev/null
    ```

4. [**user**@**murmurd-dvm**]() Prevent persistent bash history in `.zsh` _(recommended)_

    ```bash
    echo 'rm -f "$HISTFILE"' >> ~/.zshrc
    ```

5. [**user**@**murmurd-dvm**]() Bind Mumble server config and database

    <!-- 
    TODO: This step needs testing
     -->

    <details>
    <summary><b>Step Details</b></summary>

    > This will make your `mumble-server.ini` and `mumble-server.sqlite` files persist in the _DispVM_ template. If you do not do this, your server's settings, channels and registered users will be erased.

    </details>

    1. Create `qubes-bind-dirs.d` directory

        ```bash
        sudo mkdir -p /rw/config/qubes-bind-dirs.d
        ```

    2. Create and populate user bindings config

        ```bash
        echo "binds+=( '/etc/mumble-server.ini' '/var/lib/mumble-server/mumble-server.sqlite' )" | sudo tee -a /rw/config/qubes-bind-dirs.d/50_user.conf &>/dev/null
        ```

    3. Bind Mumble server database directory

        ```bash
        sudo mkdir -p /rw/bind-dirs/var/lib/mumble-server
        ```

    4. Create placeholder for Mumble sqlite database

        <details>
        <summary><b>Step Details</b></summary>

        > The `mumble-server.sqlite` database does not exist yet, so we must initialize it via the `bind-dir` directory and mirror the expected ownership and file permissions.

        </details>

        1. Create empty file
        
            ```bash
            sudo touch /rw/bind-dirs/var/lib/mumble-server/mumble-server.sqlite
            ```

        2. Set expected ownership _(`mumble-server`)_

            ```bash
            sudo chown mumble-server: /rw/bind-dirs/var/lib/mumble-server/mumble-server.sqlite
            ```

        3. Set expected permissions _(`rw-r-----`)_

            ```bash
            sudo chmod 640 /rw/bind-dirs/var/lib/mumble-server/mumble-server.sqlite
            ```
    
    3. Shutdown the VM to finalize bindings

        ```bash
        sudo poweroff
        ```

6. [**user**@**dom0**]() Start `murmurd-dvm` up again

    ```bash
    setsid qvm-run murmurd-dvm xfce4-terminal &>/dev/null
    ```

7. [**user**@**murmurd-dvm**]() Create `whonix_firewall.d` directory

    ```bash
    sudo mkdir -p /usr/local/etc/whonix_firewall.d
    ```

8. [**user**@**murmurd-dvm**]() Open ports for Mumble server

    ```bash
    echo 'EXTERNAL_OPEN_PORTS+=" 64738 "' | sudo tee -a /usr/local/etc/whonix_firewall.d/50_user.conf &>/dev/null
    ```

9. [**user**@**murmurd-dvm**]() Reload Whonix Firewall

    ```bash
    sudo whonix_firewall
    ```

10. [**user**@**murmurd-dvm**]() Reconfigure Mumble Server

    <details>
    <summary><b>Recommendations</b></summary>

    > - Autostart is suggested, or run `sudo service mumble-server start` _(Maybe test with no autostart first)_
    > - When prompted `Higher Priority?` select `Yes`
    > - Choose a secure password _(admin password?)_

    </details>

    ```bash
    sudo dpkg-reconfigure mumble-server
    ```

11. [**user**@**murmurd-dvm**]() Update Mumble server config

    <details>
    <summary><b>Recommendations</b></summary>

    > - Set the `serverpassword` to something secure, e.g., `serverpassword=DTGCEK7Qq8Zon6Z`
    > - Consider setting `logfile` to empty to prevent any service logs for better security
    > - Disable `allowping` setting because it exposes server information _(maybe not needed?)_
    > - See additional settings in this archived [Murmurd guide↗](https://archive.ph/Vvm2C)

    </details>

    ```bash
    sudo vim /etc/mumble-server.ini
    ```
12. [**user**@**murmurd-dvm**]() Shutdown VM

    ```bash
    sudo poweroff
    ```

13. [**user**@**dom0**]() Create Mumble Server _DispVM_ _(`murmurd`)_

    > <picture>
    >   <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/light-theme/note.svg">
    >   <img alt="Note" src="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/dark-theme/note.svg">
    > </picture><br>
    >
    > This is a _named_ disposable VM. Unlike standard _DispVMs_, a _named_ disposable maintains a constant _internal IP_, which we need in order to configure a Tor hidden service.

    > <picture>
    >   <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/light-theme/note.svg">
    >   <img alt="Note" src="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/dark-theme/note.svg">
    > </picture><br>
    >
    > This is the only VM which can run our Mumble server because it has the `persistent-none` flag, which we made a requirement for the `mumble-server` systemd service.

    ```bash
    qvm-create murmurd -t murmurd-dvm -l purple --class DispVM --prop netvm=sys-whonix
    ```

<br>

## 3. Configure Tor Hidden Service For Mumble Server

1. [**user**@**dom0**]() Open a terminal in `sys-whonix`

    ```bash
    setsid qvm-run sys-whonix xfce4-terminal &>/dev/null
    ```

2. [**user**@**sys-whonix**]() Create a new hidden service in `torrc` _(user config)_

    > <picture>
    > <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/light-theme/warning.svg">
    > <img alt="Warning" src="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/dark-theme/warning.svg">
    > </picture><br>
    >
    > Make sure to replace **`MURMURD_QUBE_INTERNAL_IP`** with the _internal ip_ of the `murmurd` _DispVM_.
    >
    > <details>
    > <summary><b>Where can I find the internal IP?</b></summary>
    >
    > > - **Via [user@dom0]()**
    > >
    > >     ```bash
    > >     qvm-prefs murmurd visible_ip
    > >     ```
    > > - **Via [user@murmurd]()**
    > >
    > >     ```bash
    > >     qubesdb-read /qubes-ip
    > >     ```
    >
    > </details>

    ```bash
    cat << 'EOF' | sudo tee -a /usr/local/etc/torrc.d/50_user.conf &>/dev/null

    HiddenServiceDir /var/lib/tor/mumble-server
    HiddenServicePort 64738 MURMURD_QUBE_INTERNAL_IP
    EOF
    ```

3. [**user**@**sys-whonix**]() Reload Tor Daemon

    ```bash
    sudo systemctl reload tor
    ```

4. [**user**@**sys-whonix**]() Retrieve Mumble server onion address

    ```bash
    sudo cat /var/lib/tor/mumble-server/hostname
    ```

5. [**user**@**dom0**]() Start `murmurd` _DispVM_

    > <picture>
    >   <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/light-theme/note.svg">
    >   <img alt="Note" src="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/dark-theme/note.svg">
    > </picture><br>
    >
    > This will effectively start your Mumble server. Once the VM is live, you should be able to connect to the server via the `mumble-server` hidden service onion URL.

    ```bash
    setsid qvm-start murmurd &>/dev/null
    ```

<br>

# Mumble Client Setup

## 1. Create & Configure Mumble Client TemplateVM

> <picture>
>   <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/light-theme/note.svg">
>   <img alt="Note" src="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/dark-theme/note.svg">
> </picture><br>
>
> Alternatively, you can install and configure the `mumble` _apt_ package each time you start a disposable instance of `whonix-workstation-XX-dvm`.

1. [**user**@**dom0**]() Create Mumble client _TemplateVM (`mumble-ws-17`)_

    1. Clone `whonix-workstation-17`
    
        ```bash
        qvm-clone whonix-workstation-17 mumble-ws-17
        ```

    2. Update new template preferences _(recommended)_

        1. `qvm-prefs mumble-ws-17 maxmem 2000`
        2. `qvm-prefs mumble-ws-17 memory 200`

2.  [**user**@**dom0**]() Update `mumble-ws-17`

    ```bash
    qubes-vm-update --force-update --targets mumble-ws-17
    ```

3.  [**user**@**dom0**]() Launch `mumble-ws-17`

    ```bash
    setsid qvm-run mumble-ws-17 xfce4-terminal &>/dev/null
    ```

4.  [**user**@**mumble-ws-17**]() Install _apt_ package `mumble`

    ```bash
    sudo apt install -y mumble
    ```

5. [**user**@**mumble-ws-17**]() Shutdown template

    ```bash
    sudo poweroff
    ```

<br>

## 2. Create & Configure Mumble Client DispVM

1. [**user**@**dom0**]() Create Mumble client _DispVM_

    ```bash
    qvm-create mumble-dvm -t mumble-ws-17 -l red --prop netvm=sys-whonix --prop template_for_dispvms=true --prop default_dispvm=mumble-dvm
    ```

2. [**user**@**dom0**]() Enable feature `appmenus-dispvm`

    ```bash
    qvm-features mumble-dvm appmenus-dispvm 1
    ```

3. [**user**@**dom0**]() Set label to _purple_

    <details>
    <summary><b>Step Details</b></summary>

    > There is a bug _(or feature)_ in Qubes where configuring a _DispVM_ which derives its _disposable template_ from itself does not trigger the label icon to change to the _DispVM_ icon. A workaround is to set the label color _after_ creating the VM, which refreshes the icon.
    
    </details>

    ```bash
    qvm-prefs mumble-dvm label purple
    ```

4. [**user**@**dom0**]() Refresh the _app menu_ of `mumble-dvm`

    <details>
    <summary><b>Step Details</b></summary>

    > This is requires, otherwise the _app menu_ of `mumble-dvm` will be empty. This is less a bug and more the result of using `qvm-create` and `qvm-prefs` instead of the more technical [Salt↗](https://www.qubes-os.org/doc/salt/) method.
    
    </details>

    ```bash
    qvm-appmenus --update mumble-dvm
    ```

_todo: add configuration persistence steps_

<br>

## Optimal Mumble Client Settings

- **Audio Input**

    - **Interface**

        - `Echo Cancellation`: _Mixed echo cancellation_

    - **Transmission**

        - **Amplitude**

            - `Voice Hold`: _250ms_

            - `Silence Below`: _~30% (may depend on input device)_

            - `Speech Above`: _~50% (may depend on input device)_

    - **Compression**

        - `Quality`: _~13 kb/s_

        - `Audio per packet`: _20ms_

    - **Audio Processing**

        - `Noise Suppression`: _-60 dB (may depend on input device)_

        - `Max. Amplification`: _~2.5  (may depend on input device)_

- **Audio Output**

    - **Audio Output**

        - `Default Jitter Buffer`: _10ms_

        - `Volume`: _Recommended that you start very low (10%) and adjust volume of each user manually_

            <details>
            <summary><b>How do I adjust volume for each user?</b></summary>
            
            > Once you've connected to a server, `Right-Click` the user you want to adjust and select `Local Volume Adjustment`.
            
            </details>

        - `Output Delay`: _20ms-40ms_

- **Network**

    - **Connection**

        - `Force TCP mode`: _Enabled_

            > <picture>
            > <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/light-theme/warning.svg">
            > <img alt="Warning" src="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/dark-theme/warning.svg">
            > </picture><br>
            >
            > You must enable _TCP mode_, otherwise Mumble will not work. This is because Tor only works over TCP _(confirmed packets)_, while most VOIP protocols communicate over UDP _(streamed packets)_.

        - `Reconnect automatically`: _Disabled (There is no way to stop Mumble from retrying if enabled)_

    - **Proxy**

        > <picture>
        > <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/light-theme/warning.svg">
        > <img alt="Warning" src="https://raw.githubusercontent.com/Mqxx/GitHub-Markdown/main/blockquotes/badge/dark-theme/warning.svg">
        > </picture><br>
        >
        > While not mandatory, by not applying these proxy settings, your Mumble client will be conntected via Tor's transproxy _(127.0.0.1:9050)_ in `sys-whonix`, which means it will share a Tor circuit with other apps communicating via the transproxy.
        >
        > Instead, an isolated _SocksPort_ can be used; in this case is the first _custom_ Whonix SocksPort _(9180)_, which isolates unique destination addresses and ports to their own circuit.

        - `Type`: _SOCKS5 proxy_
        
        - `Hostname`: _sys-whonix internal ip_

        - `Port`: _9180 (Fully Isolated SocksPort)_

            <details>
            <summary><b>Where can I find the internal IP?</b></summary>
            
            > - **Via [user@dom0]()**
            >
            >     ```bash
            >     qvm-prefs sys-whonix visible_ip
            >     ```
            > - **Via [user@sys-whonix]()**
            >
            >     ```bash
            >     qubesdb-read /qubes-ip
            >     ```
            
            </details>
    
    - **Privacy**

        - `Do not send OS information`: _Enabled_

    - **Mumble services**

        - `Submit anonymous statistics`: _Disabled_

<br>

# Resources

- https://www.whonix.org/wiki/Voip#Mumble_Server_Instructions
- https://forum.qubes-os.org/t/exposing-mumble-server-running-in-qubes-using-wireguard
- https://github.com/Whonix/anon-gw-anonymizer-config/blob/master/usr/share/tor/tor-service-defaults-torrc.anondist
- https://www.blunix.com/blog/creating-qubes-os-vms-on-the-command-line.html
