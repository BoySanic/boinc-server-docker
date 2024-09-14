# the tag this Dockerfile will build, either "-b2d" or ""
ARG TAG

#===============================
FROM debian:bookworm-slim AS base
#===============================

LABEL maintainer="BoySanic <boysanic@0xa.pw>"


# install packages
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        dh-autoreconf \
        g++ \
        git \
        libcurl4-gnutls-dev \
        default-libmysqlclient-dev \
        libssl-dev \
        m4 \
        make \
        default-mysql-client \
        pkg-config \
        python-is-python3 \
        python3 \
        python3-pip \
        python3-dev

#RUN curl -sSL https://packages.sury.org/php/README.txt | bash -x
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        php8.2-cli \
        php8.2-mysql \
        php8.2-xml \
    && rm -rf /var/lib/apt/lists


# get source and compile server
COPY --chown=1000 boinc /usr/local/boinc
RUN cd /usr/local/boinc && ./_autosetup && ./configure --disable-client --disable-manager && make

# project-making scripts
COPY makeproject-step1.sh makeproject-step2.sh /usr/local/bin

# some other project files (some of which will be put in the correct place with
# ONBUILD instructions later)
COPY --chown=1000 db_dump_spec.xml /.project_root/
COPY --chown=1000 html /.project_root/html/
COPY --chown=1000 secrets.env /run/secrets/


#===========================
FROM base$TAG AS makeproject
#===========================

# everything which depends on build-args is done as ONBUILD in this stage, so
# the user can customize it

ARG TAG
ENV TAG=$TAG
ONBUILD ARG BOINC_USER
ONBUILD ARG PROJECT_ROOT
ONBUILD ENV BOINC_USER=$BOINC_USER \
            PROJECT_ROOT=$PROJECT_ROOT \
            USER=$BOINC_USER \
            HOME=/home/$BOINC_USER \
            MYSQL_HOST=mysql

# set up the non-root user who compiles the server
ONBUILD RUN adduser $BOINC_USER --disabled-password --gecos ""

# ensure the project/secrets volumes have the right permissions when mounted
ONBUILD RUN mkdir -p $PROJECT_ROOT.dst && chown -R $BOINC_USER:$BOINC_USER $PROJECT_ROOT.dst/.. && chmod -R 770 $PROJECT_ROOT.dst/..
ONBUILD RUN pip3 install mysqlclient --break-system-packages
ONBUILD USER $BOINC_USER

# build server
ONBUILD RUN makeproject-step1.sh
#ONBUILD RUN test -z "$TAG" || boinc2docker_create_app --projhome $PROJECT_ROOT
ONBUILD CMD makeproject-step2.sh

ONBUILD WORKDIR $PROJECT_ROOT



#==========================================
FROM makeproject AS makeproject-defaultargs
#==========================================

# this triggers the ONBUILD directives using the default ARGs so we also get a
# fully built example image