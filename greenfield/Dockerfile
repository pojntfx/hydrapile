FROM debian:bookworm AS build

RUN apt update
RUN apt install -y autoconf libtool automake pipx ninja-build bison cmake build-essential gperf libffi-dev libudev-dev libgbm-dev libdrm-dev libegl-dev libopengl-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-bad1.0-dev libgraphene-1.0-dev supervisor npm

RUN npm i -g yarn

RUN git clone https://github.com/udevbe/greenfield.git /work

WORKDIR /work

RUN yarn install
RUN yarn workspaces foreach --parallel --topological-dev --verbose run build

FROM debian:bookworm

RUN apt update
RUN apt install -y libffi8 libudev1 libgbm1 libgraphene-1.0-0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-gl libosmesa6 libdrm2 libopengl0 libglvnd0 libglx0 libglapi-mesa libegl1-mesa libglx-mesa0 xwayland xauth xxd inotify-tools npm sudo supervisor

COPY --from=build /work /work

RUN useradd -m -s /bin/bash user
RUN echo "user ALL=(ALL) NOPASSWD:ALL" >>/etc/sudoers
ENV HOME /home/user
WORKDIR /home/user

USER user

ENV XDG_RUNTIME_DIR /home/user/.xdg-runtime-dir
RUN mkdir -p "${XDG_RUNTIME_DIR}"

ENV XAUTHORITY /tmp/.X11-unix/Xauthority

COPY supervisord.conf /etc/supervisor/supervisord.conf

RUN sudo apt install -y gnome-console gtk-4-examples libadwaita-1-examples mesa-utils weston
COPY applications.json .applications.json

CMD mkdir -p /tmp/.X11-unix && touch "$XAUTHORITY" && xauth add :1 . "$(xxd -l 16 -p /dev/urandom)" && supervisord -c /etc/supervisor/supervisord.conf
