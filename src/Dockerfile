### Base images ###

FROM opensuse/tumbleweed AS mingw64
RUN zypper --non-interactive addrepo obs://windows:mingw:win64 mingw64 && \
    zypper --non-interactive --gpg-auto-import-keys refresh
RUN zypper --non-interactive install \
        wget m4 git-core meson bsdtar gzip xmlstarlet findutils \
        mingw64-filesystem mingw64-cross-gcc mingw64-cross-gcc-c++ mingw64-cross-binutils mingw64-cross-pkgconf && \
    rpm -ql mingw64-cross-gcc &&  \
    zypper clean
RUN mkdir -p /usr/local/share/meson/cross && \
    wget -qO /usr/local/share/meson/cross/mingw64 https://raw.githubusercontent.com/mesonbuild/meson/ca30550e065991f9b2438e2a6fabc64fd2248806/cross/linux-mingw-w64-64bit.txt

FROM alpine AS imgstuff
RUN apk add --no-cache librsvg imagemagick rsvg-convert

# in alpine 3.14, wine 6.0 doesn't start for some reason
FROM i386/alpine:3.13 AS wine
ENV WINEDEBUG="-all" \
    WINEDLLOVERRIDES="mscoree,mshtml=" \
    WINEARCH="win32" \
    WINEPREFIX="/wine"
RUN apk add --no-cache wine ncurses freetype xvfb unzip wget ca-certificates

FROM wine AS reshack-install
RUN wget -qO /tmp/reshack.nupkg https://chocolatey.org/api/v2/package/reshack/5.1.8 && \
    unzip -j /tmp/reshack.nupkg tools/reshacker_setup.exe -d /tmp && \
    rm /tmp/reshack.nupkg
RUN set -x; \
    export DISPLAY=:0; \
    Xvfb :0 -screen 0 1024x768x16 & until xmodmap -display :0 &> /dev/null; do sleep .1; done && \
    wineboot --init && \
    wine "Z:\\tmp\\reshacker_setup.exe" /VERYSILENT && \
    wineboot --shutdown && \
    pkill Xvfb && while pkill -0 Xvfb &> /dev/null; do sleep .1; done

FROM wine AS reshack
RUN touch /bin/ResourceHacker && \
    chmod +x /bin/ResourceHacker && \
    echo '#!/bin/sh' >> /bin/ResourceHacker && \
    echo 'export DISPLAY=:0' >> /bin/ResourceHacker && \
    echo 'Xvfb :0 -screen 0 1024x768x16 & until xmodmap -display :0 &> /dev/null; do sleep .1; done' >> /bin/ResourceHacker && \
    echo 'wine "C:\\Program Files\\Resource Hacker\\ResourceHacker.exe" "$@"; e=$?' >> /bin/ResourceHacker && \
    echo 'pkill Xvfb && while pkill -0 Xvfb &> /dev/null; do sleep .1; done' >> /bin/ResourceHacker && \
    echo 'exit $e' >> /bin/ResourceHacker
COPY --from=reshack-install ${WINEPREFIX} ${WINEPREFIX}

FROM wine AS inno-install
RUN apk add --no-cache innoextract
RUN wget -qO /tmp/is.exe http://files.jrsoftware.org/is/6/innosetup-6.1.2.exe
RUN mkdir /opt/inno-setup && \
    innoextract --output-dir /opt/inno-setup --exclude-temp /tmp/is.exe

FROM wine AS inno
RUN touch /bin/iscc && \
    chmod +x /bin/iscc && \
    echo '#!/bin/sh' >> /bin/iscc && \
    echo 'wine /opt/inno-setup/app/ISCC.exe "$@"' >> /bin/iscc && \
    echo 'exit $?' >> /bin/iscc
COPY --from=inno-install /opt/inno-setup /opt/inno-setup

### Icon ###

FROM imgstuff AS build-icon
COPY ./icon/* /src/
RUN set -x; \
    mkdir -p /build && \
    rsvg-convert -w 16  -h 16  /src/pulseaudio-flat-simple.svg > /build/pulseaudio-flat.16.png && \
    rsvg-convert -w 32  -h 32  /src/pulseaudio-flat-simple.svg > /build/pulseaudio-flat.32.png && \
    rsvg-convert -w 48  -h 48  /src/pulseaudio-flat-simple.svg > /build/pulseaudio-flat.48.png && \
    rsvg-convert -w 64  -h 64  /src/pulseaudio-flat.svg > /build/pulseaudio-flat.64.png && \
    rsvg-convert -w 96  -h 96  /src/pulseaudio-flat.svg > /build/pulseaudio-flat.96.png && \
    rsvg-convert -w 128 -h 128 /src/pulseaudio-flat.svg > /build/pulseaudio-flat.128.png && \
    convert /build/pulseaudio-flat.*.png /pulseaudio.ico

### PulseAudio ###

FROM mingw64 AS build-pulseaudio-stage1
RUN zypper --non-interactive install \
        intltool gettext-tools mingw64-libsndfile-devel mingw64-libintl-devel mingw64-win_iconv-devel mingw64-pcre-devel \
        orc mingw64-libspeex-devel mingw64-liborc-devel mingw64-libtool && \
    zypper clean
RUN mkdir -p /src
RUN git init /src/pulseaudio && \
    git -C /src/pulseaudio remote add origin https://gitlab.freedesktop.org/pulseaudio/pulseaudio.git && \
    git -C /src/pulseaudio fetch origin && \
    git -C /src/pulseaudio checkout stable-16.x
COPY ./pulseaudio/* /src/
### Manually patch
#RUN git -C /src/pulseaudio apply /src/*.patch
RUN git -C /src/pulseaudio apply /src/0001-client-conf-Enable-auto-connect-localhost-by-default.patch
RUN git -C /src/pulseaudio apply /src/0002-system.pa-Tweaks-for-custom-build.patch
RUN git -C /src/pulseaudio apply /src/0003-system.pa-Tweaks-for-custom-build-with-unix-sockets.patch
# GIO already disabled by default
#RUN git -C /src/pulseaudio apply /src/0004-build-sys-meson-Fix-building-without-GIO.patch
ARG PAW32_VERSION=unknown
RUN git -C /src/pulseaudio describe --always --tags | \
    sed -e "s/^v//g" -e "s/$/-paw32-${PAW32_VERSION}/g" | \
    tee /src/pulseaudio/.tarball-version /src/pulseaudio/.version /pulseaudio.version
RUN meson setup \
        --buildtype plain \
        --cross-file mingw64 \
        -Dalsa=disabed \
        -Ddoxygen=disabled \
        -Dgsettings=disabled  \ 
        -Dgstreamer=disabled \
        -Dgtk=disabled \
        -Djack=disabled \
        -Dsamplerate=disabled \
        -Dlirc=disabled \
        -Dorc=disabled \
        -Doss-output=disabled \
        -Dopenssl=disabled \
        -Dtests=false \
        -Dudev=disabled \
        -Dwebrtc-aec=disabled \
        -Davahi=disabled \
        -Dfftw=disabled \
        -Dsoxr=disabled \ 
        -Dglib=disabled \
        -Dasyncns=disabled \ 
        -Delogind=disabled \
        -Dx11=disabled \
        -Dsystemd=disabled \
        -Dipv6=false
        -Ddbus=disabled \
        -Dtests=false \
        -Ddatabase=simple \
        -Ddoxygen=false \
        -Drunning-from-build-tree=false \
        -Dlegacy-database-entry-format=false \
        -Dprefix=/pulseaudio \
        -Dmodlibexecdir=/pulseaudio/bin \
        -Dbluez5=disabled \
        # _WIN32_WINNT=_WIN32_WINNT_WIN7
        -Dc_args="-D__USE_MINGW_ANSI_STDIO=1 -D_WIN32_WINNT=0x0601" \
        /build/pulseaudio /src/pulseaudio
RUN ninja -C /build/pulseaudio
RUN meson install -C /build/pulseaudio
RUN find /pulseaudio \
        # static libraries
        -name '*.a' \
        -delete && \
    rm -r \
        # development files
        /pulseaudio/lib/pkgconfig /pulseaudio/include /pulseaudio/lib/cmake /pulseaudio/share/vala \
        # manual pages
        /pulseaudio/share/man \
        # shell completion
        /pulseaudio/share/bash-completion /pulseaudio/share/zsh \
        # extra locales
        /pulseaudio/share/locale \
        # misc scripts (not compatible with windows)
        /pulseaudio/bin/pa-info && \
    rmdir /pulseaudio/share /pulseaudio/lib
RUN bash /src/mingw64-deps.sh "https://download.opensuse.org/repositories/windows:/mingw:/win32/openSUSE_Tumbleweed" /pulseaudio/bin

FROM mingw64 AS build-res
COPY --from=build-pulseaudio-stage1 /pulseaudio.version /pulseaudio.version
RUN touch /tmp/pulseaudio.rc && \
    echo '#include "winver.h"' >> /tmp/pulseaudio.rc && \
    echo 'VS_VERSION_INFO VERSIONINFO {' >> /tmp/pulseaudio.rc && \
    echo '    BLOCK "StringFileInfo" {' >> /tmp/pulseaudio.rc && \
    echo '        BLOCK "040904b0" {' >> /tmp/pulseaudio.rc && \
    echo '            VALUE "CompanyName", "Patrick Gaskin\0"' >> /tmp/pulseaudio.rc && \
    echo '            VALUE "ProductName", "PulseAudio\0"' >> /tmp/pulseaudio.rc && \
    echo '            VALUE "ProductVersion", "pulseaudio-win32 '"$(cat /pulseaudio.version)"'\0"' >> /tmp/pulseaudio.rc && \
    echo '            VALUE "FileDescription", "PulseAudio\0"' >> /tmp/pulseaudio.rc && \
    echo '            VALUE "FileVersion", "0.0.0.0\0"' >> /tmp/pulseaudio.rc && \
    echo '            VALUE "OriginalFilename", "pulseaudio.exe\0"' >> /tmp/pulseaudio.rc && \
    echo '            VALUE "InternalName", "pulseaudio\0"' >> /tmp/pulseaudio.rc && \
    echo '        }' >> /tmp/pulseaudio.rc && \
    echo '    }' >> /tmp/pulseaudio.rc && \
    echo '    BLOCK "VarFileInfo" {' >> /tmp/pulseaudio.rc && \
    echo '        VALUE "Translation", 0x409, 1200' >> /tmp/pulseaudio.rc && \
    echo '    }' >> /tmp/pulseaudio.rc && \
    echo '}' >> /tmp/pulseaudio.rc
# note: it needs to be in a subdir (/tmp) or windres gives an assertion error about include directories
RUN x86_64-w64-mingw32-windres /tmp/pulseaudio.rc -o /pulseaudio.res

FROM reshack AS build-pulseaudio-stage2
COPY --from=build-pulseaudio-stage1 /pulseaudio         /pulseaudio
COPY --from=build-pulseaudio-stage1 /pulseaudio.version /pulseaudio.version
COPY --from=build-res               /pulseaudio.res     /pulseaudio.res
COPY --from=build-icon              /pulseaudio.ico     /pulseaudio.ico
RUN ResourceHacker \
        -open "Z:\\pulseaudio\\bin\\pulseaudio.exe" \
        -save "Z:\\pulseaudio\\bin\\pulseaudio.exe" \
        -action add \
        -res "Z:\\pulseaudio.res"
RUN ResourceHacker \
        -open "Z:\\pulseaudio\\bin\\pulseaudio.exe" \
        -save "Z:\\pulseaudio\\bin\\pulseaudio.exe" \
        -action addskip \
        -res "Z:\\pulseaudio.ico" \
        -mask ICONGROUP,MAINICON,

FROM mingw64 AS build-pulseaudio
COPY --from=build-pulseaudio-stage2 /pulseaudio         /pulseaudio
COPY --from=build-pulseaudio-stage2 /pulseaudio.version /
RUN bsdtar -acf /pulseaudio.zip -C / pulseaudio

### pulseaudio (docs) ###

FROM mingw64 as build-padoc
COPY --from=build-pulseaudio-stage1 /build/pulseaudio/man /build/pulseaudio/man
COPY --from=build-pulseaudio-stage1 /src/pulseaudio/man   /src/pulseaudio/man
RUN zypper --non-interactive install libxslt-tools && \
    zypper clean
RUN mkdir -p /padoc && \
    for f in /build/pulseaudio/man/*.xml; do \
        echo "$f"; \
        xsltproc --output "/padoc/doc/$(basename "${f%.xml}.html")" --path /src/pulseaudio/man /src/pulseaudio/man/xmltoman.xsl "$f"; \
    done
RUN bsdtar -acf /padoc.zip -C / padoc

### pasvc ###

FROM mingw64 AS build-pasvc
COPY ./pasvc/* /src/
RUN mkdir -p /pasvc/bin
ARG PAW32_VERSION=unknown
RUN x86_64-w64-mingw32-gcc -Wall -Wextra -Werror -D_WIN32_WINNT=_WIN32_WINNT_VISTA -DPAW32_VERSION='"'"${PAW32_VERSION}"'"' /src/pasvc.c   -o /pasvc/bin/pasvc.exe   -lshlwapi -lshell32
RUN x86_64-w64-mingw32-gcc -Wall -Wextra -Werror -D_WIN32_WINNT=_WIN32_WINNT_VISTA -DPAW32_VERSION='"'"${PAW32_VERSION}"'"' /src/pasvcfw.c -o /pasvc/bin/pasvcfw.exe -lshlwapi -lole32 -loleaut32
RUN bsdtar -acf /pasvc.zip -C / pasvc

### pasetup ###

FROM inno AS build-pasetup
COPY --from=build-pulseaudio /pulseaudio         /src/pulseaudio
COPY --from=build-pulseaudio /pulseaudio.version /src/pulseaudio.version
COPY --from=build-padoc      /padoc              /src/padoc
COPY --from=build-pasvc      /pasvc              /src/pasvc
COPY ./pasetup/pasetup.iss                       /src/pasetup.iss
RUN iscc /O"Z:\\" /DAppVersion="$(cat /src/pulseaudio.version)" "Z:\\src\\pasetup.iss"

### output ###

FROM scratch AS output-merge
COPY --from=build-pulseaudio /pulseaudio.zip     /
COPY --from=build-pulseaudio /pulseaudio.version /
COPY --from=build-padoc      /padoc.zip          /
COPY --from=build-pasvc      /pasvc.zip          /
COPY --from=build-pasetup    /pasetup.exe        /
COPY --from=build-icon       /pulseaudio.ico     /

FROM scratch AS output
COPY --from=output-merge / /
