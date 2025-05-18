MUMBLE SERVER IN QUBES


TODO: Figure out how to do this all in AppVM based on WS-17 via /rw


1.  [**user**@**dom0**]() Create Mumble Server Template _(`murmurd-ws-17`)_

    1. Clone `whonix-workstation-17`
    
        ```bash
        qvm-clone whonix-workstation-17 murmurd-ws-17
        ```

    2. Update new template preferences

        1. `qvm-prefs murmurd-ws-17 include_in_backups False` _(optional)_
        2. `qvm-prefs murmurd-ws-17 maxmem 2000`
        3. `qvm-prefs murmurd-ws-17 memory 200`

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
    cat << 'EOF' > /etc/systemd/system/mumble-server.service.d/override.conf
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

8.  [**user**@**dom0**]() Create Mumble Server _AppVM_ _(`murmurd-dvm`)_

    <details>
    <summary><b>Step Details</b></summary>

    > This VM is setup as a _DispVM template_ for the named _DispVM_ that'll be running the server.

    </details>

    ```bash
    qvm-create murmurd-dvm -t murmurd-ws-17 -l black --prop netvm='' --prop template_for_dispvms=True --prop default_dispvm=''
    ```

9.  [**user**@**dom0**]() Enable `appmenus-dispvm` feature in `murmurd-dvm`

    ```bash
    qvm-features murmurd-dvm appmenus-dispvm 1
    ```

10. [**user**@**dom0**]() Launch `murmurd-dvm`

    ```bash
    setsid qvm-run murmurd-dvm xfce4-terminal &>/dev/null
    ```

11. [**user**@**murmurd-dvm**]() Prevent persistent bash history in `.zsh` _(recommended)_

    ```bash
    echo 'rm -f "$HISTFILE"' >> ~/.zshrc
    ```

12. [**user**@**murmurd-dvm**]() Bind Mumble server config and database

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

    4. Create placeholder for Mumble database

        ```bash
        sudo touch /rw/bind-dirs/var/lib/mumble-server/mumble-server.sqlite
        ```
    
    3. Shutdown the VM to complete bindings

        ```bash
        sudo poweroff
        ```

13. [**user**@**dom0**]() Start `murmurd-dvm` up again

    ```bash
    setsid qvm-run murmurd-dvm xfce4-terminal &>/dev/null
    ```

14. [**user**@**murmurd-dvm**]() Create `whonix_firewall.d` directory

    ```bash
    sudo mkdir -p /usr/local/etc/whonix_firewall.d
    ```

15. [**user**@**murmurd-dvm**]() Open ports for Mumble server

    ```bash
    echo 'EXTERNAL_OPEN_PORTS+=" 64738 " | sudo tee /usr/local/etc/whonix_firewall.d/50_user.conf &>/dev/null
    ```

16. [**user**@**murmurd-dvm**]() Reload Whonix Firewall

    ```bash
    sudo whonix_firewall
    ```

17. [**user**@**murmurd-dvm**]() Reconfigure Mumble Server

    <details>
    <summary><b>Recommendations</b></summary>

    > - Autostart is suggested, or run `sudo service mumble-server start` _(Maybe test with no autostart first)_
    > - When prompted `Higher Priority?` select `Yes`
    > - Choose a secure password _(admin password?)_

    </details>

    ```bash
    sudo dpkg-reconfigure mumble-server
    ```

18. [**user**@**murmurd-dvm**]() Update Mumble server config

    <details>
    <summary><b>Recommendations</b></summary>

    > - Set the `serverpassword` to something secure, e.g., `serverpassword=DTGCEK7Qq8Zon6Z`
    > - Consider setting `logfile` to empty to prevent any service logs for better security
    > - Disable `allowping` setting because it exposes server information _(maybe not needed?)_
    > - See additional settings in this archived [Murmurd guide](https://archive.ph/Vvm2C)

    </details>

    ```bash
    sudo vim /etc/mumble-server.ini
    ```


X.  [dom0]() Create Mumble Server Named DispVM _(`murmurd`)_

    ```bash
    qvm-create murmurd --class DispVM -l purple -t murmurd-dvm
    ```
