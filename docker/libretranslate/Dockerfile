# LibreTranslate with gunicorn's worker recycling turned off.
#
# The official image starts gunicorn with "--max-requests 250 --timeout ${LT_TIMEOUT:-2400}".
# Once a worker has served 250 requests it logs "Autorestarting worker after current request"
# and tries to exit -- but it never does: it blocks forever on a futex, holding a lock the
# native translation libraries (ctranslate2/OpenMP) left behind in the child at fork time.
# The process stays alive and listening but answers nothing, and the arbiter only replaces it
# once the request timeout expires, which defaults to 2400s. With both workers in that state
# the container reads "Up" in docker ps while every translation times out, for 40 minutes at a
# stretch. Dropping max-requests removes the trigger; LT_TIMEOUT in compose shortens the
# recovery if anything else ever wedges a worker.
FROM libretranslate/libretranslate:latest

USER root
RUN sed -i 's/--max-requests 250/--max-requests 0/' /app/scripts/entrypoint.sh \
    && grep -q -- '--max-requests 0' /app/scripts/entrypoint.sh
USER libretranslate
