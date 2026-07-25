# Builds a CRM image that actually contains the crm app.
#
# Why this exists: the published ghcr.io/frappe/crm images (both :stable and
# :main) ship with only `frappe` in apps/ -- the crm app is missing entirely, so
# `bench new-site --install-app crm` dies with "No module named 'crm'".
#
# The cause is upstream. .github/workflows/builds.yml passes the app list as an
# APPS_JSON_BASE64 *build arg*, but frappe_docker's images/layered/Containerfile
# now reads it from a BuildKit *secret* (--mount=type=secret,id=apps_json).
# The build arg is silently ignored, `bench init` runs with no --apps_path, and
# the result is a frappe-only image wearing a crm name.
#
# Rather than reproduce frappe_docker's whole build (its final stage COPYs
# entrypoint scripts from that repo's context), this layers the app onto the
# published image, which already carries a correct bench, nginx-entrypoint.sh
# and entrypoint.sh.

ARG BASE_IMAGE=ghcr.io/frappe/crm:stable
FROM ${BASE_IMAGE}

# Point these at a fork/branch to deploy your own code instead of upstream main.
ARG CRM_REPO=https://github.com/frappe/crm
ARG CRM_BRANCH=main

USER frappe
WORKDIR /home/frappe/frappe-bench

# The base image bakes assets at /home/frappe/frappe-bench/assets and symlinks
# sites/assets to it at runtime (see /usr/local/bin/entrypoint.sh), because
# sites/ is a volume and would otherwise shadow them. CRM's freshly built assets
# have to be folded into that baked directory or they vanish on first boot.
RUN bench get-app --branch "${CRM_BRANCH}" --resolve-deps --skip-assets crm "${CRM_REPO}" \
    && bench build --app crm \
    && if [ -d sites/assets ] && [ ! -L sites/assets ]; then \
         cp -a sites/assets/. assets/ && rm -rf sites/assets; \
       fi \
    && find apps -mindepth 1 -path "*/.git" -prune -exec rm -rf {} + \
    && rm -rf /home/frappe/.cache/yarn /home/frappe/.cache/pip

USER frappe
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["start.sh"]
