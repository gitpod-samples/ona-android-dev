FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04
# RUN useradd -u 3333 -G sudo -m -d /home/gitpod -s /bin/bash gitpod && echo "gitpod:gitpod" | chpasswd \
#     && sed -i.bkp -e '/Defaults\tuse_pty/d' -e 's/%sudo\s\+ALL=(ALL\(:ALL\)\?)\s\+ALL/%sudo ALL=NOPASSWD:ALL/g' /etc/sudoers
USER vscode
ENV HOME=/home/vscode
# SHELL ["/bin/bash", "-c"]

# # use this Dockerfile to install additional tools you might need, e.g.
# # RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
# #     && apt-get -y install --no-install-recommends <your-package-list-here>

ENV ANDROID_HOME=$HOME/android-sdk
ENV PATH="$ANDROID_HOME/emulator:$ANDROID_HOME/tools:$ANDROID_HOME/tools/bin:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/cmdline-tools/bootstrap/bin:$ANDROID_HOME/platform-tools:$PATH"
ENV PATH="$HOME/.local/bin:$HOME/.bun/bin:$PATH"

# Install Android SDK
RUN sudo apt-get update && export DEBIAN_FRONTEND=noninteractive \
    && sudo apt-get -y install --no-install-recommends \
        curl \
        unzip \
        openjdk-21-jdk

RUN version="9477386" && mkdir -p $ANDROID_HOME/cmdline-tools/bootstrap \
    && curl -L https://dl.google.com/android/repository/commandlinetools-linux-${version}_latest.zip -o /tmp/cmdline-tools.zip \
    && unzip -q /tmp/cmdline-tools.zip -d /tmp/unzip && mv /tmp/unzip/*/* $ANDROID_HOME/cmdline-tools/bootstrap

RUN yes | sdkmanager --licenses \
    && sdkmanager "cmdline-tools;latest" \
    && sdkmanager \
        "platform-tools" \
        "build-tools;35.0.0" \
        "ndk;28.1.13356709" \
        "platforms;android-35" \
        "emulator" \
        "add-ons;addon-google_apis-google-24" \
        "cmake;4.0.2" \
        "system-images;android-35;google_apis;x86_64"

# Cold boot the emulator to start it quickly later
RUN avdmanager create avd --name headless_avd --package "system-images;android-35;google_apis;x86_64" --device "pixel" && avdmanager list avd
# RUN if test -e /dev/kvm; then sudo chmod 777 /dev/kvm; (until adb shell getprop sys.boot_completed | grep -q 1; do sleep 2; done && sleep 10 && adb emu kill) & emulator -avd headless_avd -no-window -no-audio -no-boot-anim -gpu swiftshader_indirect -accel off -no-metrics; fi


USER root
# Install necessary kernel tools
RUN export DEBIAN_FRONTEND=noninteractive \
    && apt-get -y install --no-install-recommends \
        kmod \
        cpu-checker \
        qemu-system-x86 qemu-utils qemu-kvm \
        file

# Install scrcpy
RUN curl -L https://github.com/Genymobile/scrcpy/releases/download/v3.2/scrcpy-linux-x86_64-v3.2.tar.gz -o /tmp/scrcpy.tar.gz \
    && tar -xf /tmp/scrcpy.tar.gz -C /tmp \
    && mv /tmp/scrcpy-linux*/scrcpy /usr/local/bin/scrcpy \
    && chmod +x /usr/local/bin/scrcpy

    
USER vscode
# Install uv & mysc
RUN curl -LsSf https://astral.sh/uv/install.sh | sh \
    && curl -fsSL https://fnm.vercel.app/install | bash
    # && sudo apt-get install -y portaudio19-dev \
    # && uv tool install mysc[web]

# # Install ws-scrcpy
# SHELL ["/bin/bash", "-lic"]
# RUN fnm install --lts \
#     && cd $HOME && git clone https://github.com/axonasif/ws-scrcpy && cd ws-scrcpy \
#     && uv run npm install && uv run npm run dist:prod
#     # && uv run bun install && uv run bun run dist:prod




# Add VNC support from https://github.com/gitpod-samples/vnc
USER root

RUN cd /tmp \
&& curl -LO https://github.com/kasmtech/KasmVNC/releases/download/v1.4.0/kasmvncserver_noble_1.4.0_amd64.deb \
&& apt-get update && apt-get install -yq ./kasm*.deb && rm kasm*.deb \
&& apt-get install -yq --no-install-recommends dbus dbus-x11 gnome-keyring xfce4 xfce4-terminal xdg-utils x11-xserver-utils

ENV DISPLAY=:1
ARG USERNAME=vscode
ARG HOMEPATH=/home/$USERNAME

RUN adduser $USERNAME ssl-cert

USER $USERNAME
SHELL ["/bin/bash", "-ic"]

RUN <<SCRIPT
    set -eu

    printf '%s\n' 123456 123456 | vncpasswd -u $USER -ow
    cat > $HOME/.xinitrc << 'EOF'
    #!/bin/bash
    
    : "${DISPLAY:="$DISPLAY"}"
    export DISPLAY
    
    exec dbus-launch --exit-with-session xfce4-session
EOF
    chmod +x $HOME/.xinitrc

    mkdir -p $HOME/.vnc
    ln -sf $HOME/.xinitrc $HOME/.vnc/xstartup
    touch $HOME/.vnc/.de-was-selected

    cat > $HOME/.vnc/kasmvnc.yaml << 'EOF'
logging:
  log_writer_name: all
  log_dest: logfile
  level: 100

network:
  protocol: http
  interface: 0.0.0.0
  websocket_port: 5901
  use_ipv4: true
  use_ipv6: false
  udp:
    public_ip: auto
    port: auto
    payload_size: auto
    stun_server: auto
  ssl:
    require_ssl: false
EOF

    sudo tee -a /usr/bin/vnc <<'BASH'
#!/bin/bash
set -eux
sudo service dbus start
cd $HOME
vncserver -disableBasicAuth -alwaysshared
BASH
    sudo chmod +x /usr/bin/vnc

SCRIPT


# Optional: Install Browser (Chrome)
# chrome and basic render font
# misc deps for electron and puppeteer to run
USER root
RUN cd /tmp && glink="https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb" \
	&& wget -q "$glink" \
	&& apt-get install -yq --no-install-recommends libasound2-dev libgtk-3-dev libnss3-dev \
	fonts-noto fonts-noto-cjk ./"${glink##*/}" \
	\
	# OLD: && ln -srf /usr/bin/chromium /usr/bin/google-chrome
	# OLD: To make ungoogled_chromium discoverable by tools like flutter
	&& ln -srf /usr/bin/google-chrome /usr/bin/chromium \
	\
	# Extra chrome tweaks
	## Disables welcome screen
	&& t="$HOMEPATH/.config/google-chrome/First Run" && sudo -u $USERNAME mkdir -p "${t%/*}" && sudo -u $USERNAME touch "$t" \
	## Disables default browser prompt
	&& t="/etc/opt/chrome/policies/managed/managed_policies.json" && mkdir -p "${t%/*}" && printf '{ "%s": %s }\n' DefaultBrowserSettingEnabled false > "$t"

# For Qt WebEngine on docker
ENV QTWEBENGINE_DISABLE_SANDBOX 1


# Install android studio

USER $USERNAME
ENV PATH="$HOMEPATH/android-studio/bin:$PATH"
SHELL ["/bin/bash", "-ic"]

RUN <<SCRIPT
    set -eu

    cd $HOME
    curl -L https://redirector.gvt1.com/edgedl/android/studio/ide-zips/2025.2.1.7/android-studio-2025.2.1.7-linux.tar.gz -o /tmp/android-studio.tar.gz
    tar -xzf /tmp/android-studio.tar.gz
    rm /tmp/android-studio.tar.gz
    # cd android-studio/bin
    # vnc
    # ./studio.sh

    # Create desktop entry
    # cat > $HOME/.local/share/applications/android-studio.desktop
SCRIPT
