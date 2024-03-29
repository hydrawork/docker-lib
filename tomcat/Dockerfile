ARG JAVA_VERSION=8

FROM hydrawork/java:${JAVA_VERSION}
LABEL maintainer="hydrawork <alex@hydra.work>"

ARG TOMCAT_REPO=http://mirrors.aliyun.com/apache/tomcat
ARG TOMCAT_MAJOR=8
ARG TOMCAT_VERSION=8.5.45
ARG CATALINA_HOME=/usr/local/tomcat

ENV CATALINA_HOME=${CATALINA_HOME} \
    PATH=${CATALINA_HOME}/bin:$PATH \
    TOMCAT_NATIVE_LIBDIR=${CATALINA_HOME}/native-jni-lib \
    LD_LIBRARY_PATH=${LD_LIBRARY_PATH:+$LD_LIBRARY_PATH:}${CATALINA_HOME}/native-jni-lib

RUN set -eux; \
    mkdir -p "$CATALINA_HOME" && \
    cd "$CATALINA_HOME" && \
	echo "TOMCAT_NATIVE_LIBDIR=$TOMCAT_NATIVE_LIBDIR" && \
	echo "PATH=$PATH" && \
	echo "TOMCAT_VERSION=$TOMCAT_VERSION" && \
	curl -o tomcat.tar.gz ${TOMCAT_REPO}/tomcat-${TOMCAT_MAJOR}/v${TOMCAT_VERSION}/bin/apache-tomcat-${TOMCAT_VERSION}.tar.gz && \
	tar -zxf tomcat.tar.gz && \
	mv apache-tomcat-$TOMCAT_VERSION/* . && \
	rmdir apache-tomcat-$TOMCAT_VERSION && \
	rm bin/*.bat && \
	rm tomcat.tar.gz* && \
	rm -rf webapps/* && \
	sed -i 's/port="8080"/port="8080" URIEncoding="UTF-8"/g' conf/server.xml && \
    \
	nativeBuildDir="$(mktemp -d)" && \
	tar -xf bin/tomcat-native.tar.gz -C "$nativeBuildDir" && \
	apk add --no-cache --virtual .native-build-deps apr-dev gcc libc-dev make openssl-dev && \
	( \
		export CATALINA_HOME="$PWD" && \
		tmp=$(ls $nativeBuildDir) && \
		cd "$nativeBuildDir/$tmp/native" && \
		./configure \
			--libdir="$TOMCAT_NATIVE_LIBDIR" \
			--prefix="$CATALINA_HOME" \
			--with-apr="$(which apr-1-config)" \
			--with-java-home="$JAVA_HOME" \
			--with-ssl=yes && \
		make -j$(getconf _NPROCESSORS_ONLN) && \
		make install \
	) && \
	runDeps="$( \
	scanelf --needed --nobanner --recursive "$TOMCAT_NATIVE_LIBDIR" \
	| awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
	| sort -u \
	| xargs -r apk info --installed \
	| sort -u \
	)" && \
	apk add --no-cache --virtual .tomcat-native-rundeps $runDeps && \
	apk del .native-build-deps curl && \
	rm -rf "$nativeBuildDir" && \
	rm bin/tomcat-native.tar.gz && \
	set -e && \
	nativeLines="$(catalina.sh configtest 2>&1)" && \
	nativeLines="$(echo "$nativeLines" | grep 'Apache Tomcat Native')" && \
	nativeLines="$(echo "$nativeLines" | sort -u)" && \
	if ! echo "$nativeLines" | grep 'INFO: Loaded APR based Apache Tomcat Native library' >&2; then \
		echo >&2 "$nativeLines"; \
		exit 1; \
	fi

WORKDIR $CATALINA_HOME

EXPOSE 8080

CMD ["catalina.sh", "run"]