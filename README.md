# openrgb-profile-at-startup
The following are instructions on how to load OpenRGB profile at startup, without modifying any default config file.

In order for this solution to take effect, you must have an `.orp` file in `~/.config/OpenRGB`.

If you don't, make sure to create a file through the GUI once, and then simply save the profile at the default location.

If you did do that, and the file still doesn't exist, open an issue & we can talk about it.

## Setup
This will create an executable called `openrgb-load-profile` at `/usr/local/bin`, which will do the heavy lifting for us.

Run the following in a `bash` shell (your distribution's default, probably):
```bash
sudo tee "/usr/local/bin/openrgb-load-profile" <<EOF
#!/usr/bin/env bash

user="\$USER"
if [[ \$(id -u) = 0 ]]; then
    user="\${SUDO_USER:-\$(logname)}"
fi

if [[ -z "\$user" ]]; then
    echo 'Could not locate calling user for openrgb-load-profile.'
    echo 'This is a fatal condition, because no profiles will be found.'
    exit 1
fi

openrgb_dir="/home/\$user/.config/OpenRGB"

if [[ ! -d "\$openrgb_dir" ]]; then
    echo "Could not locate openrgb directory \$openrgb_dir."
    echo "Have you saved/created any profile there?"
    echo "If you believe the directory exists, or should have been found elsewhere, open an issue in github."
    exit 1
fi

profile="\$openrgb_dir/\$(ls -v "\$openrgb_dir" | grep '.orp' | head -n 1)"

if [[ ! -f "\$profile" ]]; then
    echo "Could not locate any profile."
    echo "... aka, the following didn't match anything: \$profile"
    echo "If you believe the file exists, or should have been found elsewhere, open an issue on github."
    exit 1
fi

echo "Loading '\$profile'"

hasFailed="\$(openrgb --profile "\$profile"| grep -i 'failed' | grep -i 'profile')"
exit \$(test -z "\$hasFailed")
EOF
```
We need to set the correct permissions for the executable. Run:
```bash
sudo chown root:root "/usr/local/bin/openrgb-load-profile" && sudo chmod 755 "/usr/local/bin/openrgb-load-profile"
```

Now, we need to create a `systemd` service (unit file), which will load our OpenRGB script at first login.
```bash
mkdir -p ~/.config/systemd/user/ && tee "~/.config/systemd/user/openrgb.service" <<EOF
[Unit]
Description=Open source RGB lighting control that doesn't depend on manufacturer software.
After=graphical.target

[Service]
Type=oneshot
StandardOutput=journal
ExecStart=openrgb-load-profile
RemainAfterExit=yes
Restart=on-failure

[Install]
WantedBy=default.target
EOF
```

Now, we just need to enable this service.
```bash
systemctl enable --user --now openrgb.service
```
Done! The changes will take effect immediately.

You should be able to see that the service has ran by checking the `Active` section, by executing:
```bash
systemctl status --user openrgb
```
