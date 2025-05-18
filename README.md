MUMBLE SERVER IN QUBES


TODO: Figure out how to do this all in AppVM based on WS-17 via /rw


1.  [dom0] Create Mumble Server Template _(`murmurd-ws-17`)_

        qvm-clone whonix-workstation-17 murmurd-ws-17 --prop include_in_backup=False --prop memory=200 --prop maxmem=2000

2.  [dom0] Update `murmurd-ws-17`

        qubes-vm-update --force-update --targets murmurd-ws-17

3.  [dom0] Launch `murmurd-ws-17`

        setsid qvm-run murmurd-ws-17 xfce4-terminal &>/dev/null

4.  [murmurd-ws-17] Install `mumble-server` package

        sudo apt install -y mumble-server

5.  [murmurd-ws-17] Create service directory for `mumble-server`

        sudo mkdir -p /etc/systemd/system/mumble-server.service.d

6. [murmurd-ws-17] Prevent `mumble-server.service` from running in _persistent_ VMs

        ```bash
        cat << 'EOF' > /etc/systemd/system/mumble-server.service.d/override.conf
        [unit]
        ConditionPathExists=/run/qubes/persistent-none
        EOF
        ```

7.  [murmurd-ws-17] Shutdown the template

        sudo poweroff

8.  [dom0] Create Mumble Server AppVM _(`murmurd-dvm`)_

        qvm-create murmurd-dvm -t murmurd-ws-17 -l black --prop netvm=sys-whonix --prop template_for_dispvms=True --default_dispvm=None

9.  [dom0] Enable `appmenus-dispvm` feature in `murmurd-dvm`

        qvm-features murmurd-dvm appmenus-dispvm 1

10. [dom0] Launch `murmurd-dvm`

        setsid qvm-run murmurd-dvm xfce4-terminal &>/dev/null

11. [murmurd-dvm] Create `whonix_firewall.d` directory

        sudo mkdir -p /usr/local/etc/whonix_firewall.d

12. [murmurd-dvm] Open ports for Mumble server

        echo 'EXTERNAL_OPEN_PORTS+=" 64738 " | sudo tee /usr/local/etc/whonix_firewall.d/50_user.conf &>/dev/null

13. [murmurd-dvm] Reload Whonix Firewall

        sudo whonix_firewall

14. [murmurd-dvm] Reconfigure Mumble Server

    **Recommendations**
    - Autostart is suggested, or run `sudo service mumble-server start` _(Maybe test with no autostart first)_
    - When prompted `Higher Priority?` select `Yes`
    - Choose a secure password _(admin password?)_

        sudo dpkg-reconfigure mumble-server

15. [murmurd-dvm] Update Mumble server config

    **Config Settings**
    - Server password key is `serverpassword`, e.g., `serverpassword=DTGCEK7Qq8Zon6Z`
    - [Murmur Guide _(including config settings)_](https://archive.ph/Vvm2C)

        sudo nano /etc/mumble-server.ini


X.  [dom0] Create Mumble Server Named DispVM _(`murmurd`)_

        qvm-create murmurd --class DispVM -l purple -t murmurd-dvm
