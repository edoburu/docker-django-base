Base docker images for Django
=============================

These images are suitable as base for running Django inside a Docker Swarm or Kubernetes environment.

Features:

* Nessesairy base libraries to install Django.
* Multi-stage builds for smaller runtime images (below 300MB).
* No separate Nginx container is needed thanks to `uwsgi --http-socket`.
* [MozJPEG](https://github.com/mozilla/mozjpeg) installed.
* Pillow generated JPEG images pass [Google PageSpeed](https://developers.google.com/speed/pagespeed/insights/).
* Run `manage.py check --deploy` on startup.

The default ports are 8080 (HTTP) and 1717 (uWSGI stats). These higher ports allow the container to run as unprivileged user.

Static files are efficiently served from uWSGI using offload threads. You may use [WhiteNoise](http://whitenoise.evans.io/) to generate cache-busting file names, given that you have an edge-node that caches these files (e.g. AWS CloudFront). Delegating static file serving to a separate Nginx container is pretty pointless, given that everything likely runs behind an HTTP proxy already to route virtual hosts (e.g. Kubernetes Nginx ingress).

The standard libjpeg and libjpeg-turbo are optimized to quickly encode JPEG files. MozJPEG is a drop-in replacement which is slower for encoding, but faster for downloading and decoding. The resuling images are much slower and pass [Google PageSpeed](https://developers.google.com/speed/pagespeed/insights/). All images generated by Pillow, including those by [sorl-thumbnail](https://github.com/jazzband/sorl-thumbnail), use MozJPEG for rendering.

Linking Pillow against MozJPEG does require require Pillow to be installed from the source tarball using `pip install --no-binary=Pillow ...`. Otherwise, the Pillow wheel package is installed, which bundles it's own copy of libjpeg. The build image contains a caching layer for the latest Pillow image to avoid recompiling Pillow in most use-cases.

**Tips:** When using `sorl-thumbnail`, add the setting `THUMBNAIL_QUALITY = 75` to match the behavior of the default `cjpeg` tool. A quality of 80 is also sufficient. The default value for `THUMBNAIL_QUALITY` is 95, which generates overly large thumbnail files.


Available tags
--------------

Base images:

- `py36-stretch-build` ([Dockerfile](https://github.com/edoburu/docker-django-base-image/blob/master/py36-stretch-build/Dockerfile)) - Build-time container with [mozjpeg](https://github.com/mozilla/mozjpeg), and [Pillow](https://python-pillow.org/) 5.0 linked to it.
- `py36-stretch-runtime` ([Dockerfile](https://github.com/edoburu/docker-django-base-image/blob/master/py36-stretch-runtime/Dockerfile)) - Run-time container with [mozjpeg](https://github.com/mozilla/mozjpeg), and default run-time libraries.

Onbuild images:

- `py36-stretch-build-onbuild` ([Dockerfile](https://github.com/edoburu/docker-django-base-image/blob/master/py36-stretch-build/onbuild/Dockerfile)) - Pre-scripted build container that assumes `src/requirements/docker.txt` is available. Supports `PIP_REQUIREMENTS` build arg.
- `py36-stretch-runtime-onbuild` ([Dockerfile](https://github.com/edoburu/docker-django-base-image/blob/master/py36-stretch-runtime/onbuild/Dockerfile)) - Pre-scripted runtime container that assumes `src/`, `web/media` and `web/static` are available. Supports `GIT_VERSION` build arg.


Onbuild Usage
-------------

The "onbuild" images contain pre-scripted and opinionated assumptions about the application layout.
Using these images result in a very small ``Dockerfile``:

```dockerfile
FROM edoburu/django-base-images:py36-stretch-build-onbuild AS build-image

# Remove more unneeded locale files
RUN find /usr/local/lib/python3.6/site-packages/babel/locale-data/ -not -name 'en*' -not -name 'nl*' -name '*.dat' -delete && \
    find /usr/local/lib/python3.6/site-packages/tinymce/ -regextype posix-egrep -not -regex '.*/langs/(en|nl).*\.js' -wholename '*/langs/*.js' -delete

# Start runtime container
FROM edoburu/django-base-images:py36-stretch-runtime-onbuild
ENV DJANGO_SETTINGS_MODULE=mysite.settings.docker \
    UWSGI_MODULE=mysite.wsgi.docker:application

# Collect static files as root, with gzipped files for uwsgi to serve
RUN manage.py compilemessages && \
    manage.py collectstatic --settings=$DJANGO_SETTINGS_MODULE --noinput && \
    gzip --keep --best --force --recursive /app/web/static/

# Add extra uwsgi settings
COPY deployment/docker/uwsgi-local.ini /usr/local/etc/uwsgi-local.ini

# Reduce default permissions
USER app
```

The default healtcheck uses `localhost`, so adding `localhost` to `ALLOWED_HOSTS` is probably a good idea.

You may add `SILENCED_SYSTEM_CHECKS = ['security.W001']` since the `SecurityMiddleware` headers are sent by uWSGI already.


Base image usage
----------------

While the "onbuild" images are opinionated, the base images only contain what is absolutely needed. By using these images, your custom `Dockerfile` can break with all assumptions that the onbuild images make. This example is equivalent to the onbuild example above:

```dockerfile
# Build environment has gcc and develop header files.
# The installed files are copied to the smaller runtime container.
FROM edoburu/django-base-images:py36-stretch-build AS build-image

# Install (and compile) all dependencies
RUN mkdir -p /app/src/requirements
COPY src/requirements/*.txt /app/src/requirements/
ARG PIP_REQUIREMENTS=/app/src/requirements/docker.txt
RUN pip install --no-binary=Pillow -r $PIP_REQUIREMENTS

# Remove unneeded locale files
RUN find /usr/local/lib/python3.6/site-packages/ -name '*.po' -delete && \
    find /usr/local/lib/python3.6/site-packages/babel/locale-data/ -not -name 'en*' -not -name 'nl*' -name '*.dat' -delete && \
    find /usr/local/lib/python3.6/site-packages/tinymce/ -regextype posix-egrep -not -regex '.*/langs/(en|nl).*\.js' -wholename '*/langs/*.js' -delete

# Start runtime container
# Default DATABASE_URL is useful for local testing, and avoids connect timeouts for `manage.py`.
FROM edoburu/django-base-images:py36-stretch-runtime
ENV UWSGI_PROCESSES=1 \
    UWSGI_THREADS=20 \
    UWSGI_OFFLOAD_THREADS=20 \
    UWSGI_MODULE=mysite.wsgi.docker:application \
    DJANGO_SETTINGS_MODULE=mysite.settings.docker \
    DATABASE_URL=sqlite:////tmp/demo.db

# System config (done early, avoid running on every code change)
EXPOSE 8080 1717
HEALTHCHECK --interval=5m --timeout=3s CMD curl -f http://localhost:8080/api/healthcheck/ || exit 1
CMD ["/usr/local/bin/uwsgi", "--ini", "/usr/local/etc/uwsgi.ini"]
WORKDIR /app/src
VOLUME /app/web/media

# Install dependencies
COPY --from=build-image /usr/local/bin/ /usr/local/bin/
COPY --from=build-image /usr/local/lib/python3.6/site-packages/ /usr/local/lib/python3.6/site-packages/
COPY deployment/docker/manage.py /usr/local/bin/
COPY deployment/docker/uwsgi.ini /usr/local/etc/uwsgi.ini

# Insert application code.
# - Set a default database URL for accidental DB requests
# - Prepare gzipped versions of static files for uWSGI to use
# - Create a default database inside the container (as demo),
#   when caller doesn't define DATABASE_URL
# - Give full permissions, so Kubernetes can run the image as different user
COPY web /app/web
COPY src /app/src
RUN rm /app/src/*/settings/local.py* && \
    find . -name '*.pyc' -delete && \
    python -mcompileall -q */ && \
    manage.py compilemessages && \
    manage.py collectstatic --noinput && \
    gzip --keep --best --force --recursive /app/web/static/ && \
    chown -R app:app /app/web/media/ /app/web/static/CACHE && \
    chmod -R go+rw /app/web/media/ /app/web/static/CACHE

# Tag the docker image
ARG GIT_VERSION
LABEL git-version=$GIT_VERSION
RUN echo $GIT_VERSION > .docker-git-version

# Reduce default permissions
USER app
```

A `.dockerignore` with at least the following exclusions is recommended:

```
*.pyc
*.mo
*.db
*.css.map
.cache
.sass-cache
.idea
.vagrant
.git
.DS_Store
__pycache__
src/myproject/settings/local.py
src/node_modules
web/media
web/static/CACHE
```

Overriding UWSGI config
-----------------------

The onbuild image contains a default [uwsgi.ini](https://github.com/edoburu/docker-django-base-images/blob/master/py36-stretch-runtime/onbuild/uwsgi.ini) that is fully functional, but is slightly opinionated about static file paths. It can be easily extended by adding a different version:

```
COPY uwsgi-local.ini /usr/local/etc/uwsgi-local.ini
```

Or overwritten all together:

```
COPY uwsgi.ini /usr/local/etc/uwsgi.ini
```

The default [uwsgi.ini](https://github.com/edoburu/docker-django-base-images/blob/master/py36-stretch-runtime/onbuild/uwsgi.ini) serves static files entirely from uWSGI, with HTTP cache headers set. [WhiteNoise](http://whitenoise.evans.io/) can still be used to generate cache-busing file names, but serving files performs better using `uwsgi --static-map` since it can cache ``stat()`` calls and use offload threads.

