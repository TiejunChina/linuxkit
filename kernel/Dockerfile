FROM linuxkit/alpine:d307c8a386fa3f32cddda9409b9687e191cdd6f1 AS kernel-build
RUN apk add \
    argp-standalone \
    automake \
    bash \
    bc \
    binutils-dev \
    bison \
    build-base \
    curl \
    diffutils \
    flex \
    git \
    gmp-dev \
    gnupg \
    installkernel \
    kmod \
    libelf-dev \
    libressl-dev \
    linux-headers \
    mpc1-dev \
    mpfr-dev \
    ncurses-dev \
    sed \
    squashfs-tools \
    tar \
    xz \
    xz-dev \
    zlib-dev

# Update patch utility
RUN apk update && apk add --update patch

# libunwind-dev pkg is missed from arm64 now, below statement will be removed if the pkg is available.
RUN [ $(uname -m) == x86_64 ] && apk add libunwind-dev || true

ARG KERNEL_MAJOR_VERSION
ARG KERNEL_MINOR_VERSION
ARG KERNEL_SERIES
ARG EXTRA

ENV KERNEL_VERSION=${KERNEL_MAJOR_VERSION}.${KERNEL_MINOR_VERSION}

ENV KERNEL_SOURCE=https://www.kernel.org/pub/linux/kernel/v4.x/linux-${KERNEL_VERSION}.tar.xz
ENV KERNEL_SHA256_SUMS=https://www.kernel.org/pub/linux/kernel/v4.x/sha256sums.asc
ENV KERNEL_PGP2_SIGN=https://www.kernel.org/pub/linux/kernel/v4.x/linux-${KERNEL_VERSION}.tar.sign

ENV WIREGUARD_VERSION=0.0.20171221
ENV WIREGUARD_SHA256=2b97697e9b271ba8836a04120a287b824648124f21d5309170ec51c1f86ac5ed
ENV WIREGUARD_URL=https://git.zx2c4.com/WireGuard/snapshot/WireGuard-${WIREGUARD_VERSION}.tar.xz

ENV RT_PATCHES_WHERE=https://www.kernel.org/pub/linux/kernel/projects/rt/${KERNEL_MAJOR_VERSION}/
ENV RT_PATCHES_SHA256_SUMS=https://www.kernel.org/pub/linux/kernel/projects/rt/${KERNEL_MAJOR_VERSION}/sha256sums.asc

# We copy the entire directory. This copies some unneeded files, but
# allows us to check for the existence /patches-${KERNEL_SERIES} to
# build kernels without patches.
COPY / /

# Download and verify kernel
# PGP keys: 589DA6B1 (greg@kroah.com) & 6092693E (autosigner@kernel.org) & 00411886 (torvalds@linux-foundation.org)
RUN curl -fsSLO ${KERNEL_SHA256_SUMS} && \
    gpg2 -q --import keys.asc && \
    gpg2 --verify sha256sums.asc && \
    KERNEL_SHA256=$(grep linux-${KERNEL_VERSION}.tar.xz sha256sums.asc | cut -d ' ' -f 1) && \
    [ -f linux-${KERNEL_VERSION}.tar.xz ] || curl -fsSLO ${KERNEL_SOURCE} && \
    echo "${KERNEL_SHA256}  linux-${KERNEL_VERSION}.tar.xz" | sha256sum -c - && \
    xz -d linux-${KERNEL_VERSION}.tar.xz && \
    curl -fsSLO ${KERNEL_PGP2_SIGN} && \
    gpg2 --verify linux-${KERNEL_VERSION}.tar.sign linux-${KERNEL_VERSION}.tar && \
    cat linux-${KERNEL_VERSION}.tar | tar --absolute-names -x && mv /linux-${KERNEL_VERSION} /linux

WORKDIR /linux
# Download, verify and apply Preempt-RT kernel patches in the case of RT Linux Kernel
RUN if [ -n "${EXTRA}" ]; then \
	KERNEL_TYPE=$( (echo "${EXTRA}") | (grep -oE '\-rt|\-std') ); \
	if [ -z "${KERNEL_TYPE}" ]; then KERNEL_TYPE="-std"; fi; \
	echo ${KERNEL_TYPE}; \
	if [ "${KERNEL_TYPE}" = "-rt" ]; then \
	    wget ${RT_PATCHES_WHERE}; \
	    RT_PATCHES_VERSION=$( (cat index.html) | (cut -d ">" -f2) | (cut -d "<" -f1) | (grep 'tar.xz$') | (cut -d "." -f1-3) | (cut -d "-" -f2-3) ); \
	    echo ${RT_PATCHES_VERSION}; \
	    rm -rf index.html; \
	    curl -fsSLO ${RT_PATCHES_SHA256_SUMS}; \
	    gpg2 -q --import /rtkey.asc; \
	    gpg2 --verify sha256sums.asc; \
	    RT_PATCHES_SHA256=$(grep patches-${RT_PATCHES_VERSION}.tar.xz sha256sums.asc | cut -d ' ' -f 1); \
	    [ -f patches-${RT_PATCHES_VERSION}.tar.xz ] || curl -fsSLO ${RT_PATCHES_WHERE}/patches-${RT_PATCHES_VERSION}.tar.xz; \
	    echo "${RT_PATCHES_SHA256}  patches-${RT_PATCHES_VERSION}.tar.xz" | sha256sum -c -; \
	    xz -d patches-${RT_PATCHES_VERSION}.tar.xz; \
	    curl -fsSLO ${RT_PATCHES_WHERE}/patches-${RT_PATCHES_VERSION}.tar.sign; \
	    gpg2 --verify patches-${RT_PATCHES_VERSION}.tar.sign patches-${RT_PATCHES_VERSION}.tar; \
	    cat patches-${RT_PATCHES_VERSION}.tar | tar --absolute-names -x; \
	    for line in `sed -e "s/#.*//g" ./patches/series`; do \
	        patch -p1 < ./patches/$line; \
	    done; \
	fi; \
    fi

# Apply local patches if present
RUN set -e && \
    if [ -d /patches-${KERNEL_SERIES} ]; then \
        for patch in /patches-${KERNEL_SERIES}/*.patch; do \
            echo "Applying $patch"; \
            patch -p1 < "$patch"; \
        done; \
    fi

# Kernel config
RUN case $(uname -m) in \
    x86_64) \
        KERNEL_DEF_CONF=/linux/arch/x86/configs/x86_64_defconfig; \
        ;; \
    aarch64) \
        KERNEL_DEF_CONF=/linux/arch/arm64/configs/defconfig; \
        ;; \
    esac  && \
    KERNEL_TYPE=$( (echo "${EXTRA}") | (grep -oE '\-rt|\-std') ); \
    if [ -z "${KERNEL_TYPE}" ]; then KERNEL_TYPE="-std"; fi; \
    echo ${KERNEL_TYPE}; \
    if [ "${KERNEL_TYPE}" = "-std" ]; then \
	cp /config-${KERNEL_SERIES}-$(uname -m) ${KERNEL_DEF_CONF}; \
    else \
	cp /config-${KERNEL_SERIES}${KERNEL_TYPE}-$(uname -m) ${KERNEL_DEF_CONF}; \
    fi; \
    KERNEL_DEBUG=$( (echo "${EXTRA}") | (grep -oE '\-dbg') ); \
    sed -i "s/CONFIG_LOCALVERSION=\"-linuxkit\"/CONFIG_LOCALVERSION=\"-linuxkit${KERNEL_TYPE}\"/" ${KERNEL_DEF_CONF}; \
    if [ -n "${KERNEL_DEBUG}" ]; then \
        sed -i "s/CONFIG_LOCALVERSION=\"-linuxkit${KERNEL_TYPE}\"/CONFIG_LOCALVERSION=\"-linuxkit${KERNEL_TYPE}${KERNEL_DEBUG}\"/" ${KERNEL_DEF_CONF}; \
        sed -i 's/CONFIG_PANIC_ON_OOPS=y/# CONFIG_PANIC_ON_OOPS is not set/' ${KERNEL_DEF_CONF}; \
        cat /config${KERNEL_DEBUG} >> ${KERNEL_DEF_CONF}; \
    fi && \
    make defconfig && \
    make oldconfig && \
    if [ -z "${KERNEL_DEBUG}" ]; then diff .config ${KERNEL_DEF_CONF}; fi

RUN mkdir /out

# Kernel
RUN make -j "$(getconf _NPROCESSORS_ONLN)" KCFLAGS="-fno-pie" && \
    case $(uname -m) in \
    x86_64) \
        cp arch/x86_64/boot/bzImage /out/kernel; \
        ;; \
    aarch64) \
        cp arch/arm64/boot/Image.gz /out/kernel; \
        ;; \
    esac && \
    cp System.map /out && \
    ( KERNEL_DEBUG=$( (echo "${EXTRA}") | (grep -oE '\-dbg') ); \
      if [ "${KERNEL_DEBUG}" = "-dbg" ]; then cp vmlinux /out || true; fi)

# WireGuard
RUN curl -sSL -o /wireguard.tar.xz "${WIREGUARD_URL}" && \
    echo "${WIREGUARD_SHA256}  /wireguard.tar.xz" | sha256sum -c - && \
    tar -C / --one-top-level=wireguard --strip-components=2 -xJf /wireguard.tar.xz "WireGuard-${WIREGUARD_VERSION}/src" && \
    make -j "$(getconf _NPROCESSORS_ONLN)" M="/wireguard" modules

# Modules and Device Tree binaries
RUN make INSTALL_MOD_PATH=/tmp/kernel-modules modules_install && \
    make INSTALL_MOD_PATH=/tmp/kernel-modules M="/wireguard" modules_install && \
    ( DVER=$(basename $(find /tmp/kernel-modules/lib/modules/ -mindepth 1 -maxdepth 1)) && \
      cd /tmp/kernel-modules/lib/modules/$DVER && \
      rm build source && \
      ln -s /usr/src/linux-headers-$DVER build ) && \
    case $(uname -m) in \
    aarch64) \
        make INSTALL_DTBS_PATH=/tmp/kernel-modules/boot/dtb dtbs_install; \
        ;; \
    esac && \
    ( cd /tmp/kernel-modules && tar cf /out/kernel.tar . )

# Headers (userspace API)
RUN mkdir -p /tmp/kernel-headers/usr && \
    make INSTALL_HDR_PATH=/tmp/kernel-headers/usr headers_install && \
    ( cd /tmp/kernel-headers && tar cf /out/kernel-headers.tar usr )

# Headers (kernel development)
RUN DVER=$(basename $(find /tmp/kernel-modules/lib/modules/ -mindepth 1 -maxdepth 1)) && \
    dir=/tmp/usr/src/linux-headers-$DVER && \
    mkdir -p $dir && \
    cp /linux/.config $dir && \
    cp /linux/Module.symvers $dir && \
    find . -path './include/*' -prune -o \
           -path './arch/*/include' -prune -o \
           -path './scripts/*' -prune -o \
           -type f \( -name 'Makefile*' -o -name 'Kconfig*' -o -name 'Kbuild*' -o \
                      -name '*.lds' -o -name '*.pl' -o -name '*.sh' -o \
                      -name 'objtool' -o -name 'fixdep' -o -name 'randomize_layout_seed.h' \) | \
         tar cf - -T - | (cd $dir; tar xf -) && \
    ( cd /tmp && tar cf /out/kernel-dev.tar usr/src )

RUN printf "KERNEL_SOURCE=${KERNEL_SOURCE}\n" > /out/kernel-source-info

# perf (Don't compile for 4.4.x, it's broken and tedious to fix)
RUN if [ "${KERNEL_SERIES}" != "4.4.x" ]; then \
       mkdir -p /build/perf && \
       make -C tools/perf LDFLAGS=-static O=/build/perf && \
       strip /build/perf/perf && \
       cp /build/perf/perf /out; \
     fi

FROM scratch
ENTRYPOINT []
CMD []
WORKDIR /
COPY --from=kernel-build /out/* /
