FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1 PORT=8080

# Non-root user with a fixed uid, so the Kubernetes securityContext (runAsUser 10001) matches the image.
RUN useradd --system --uid 10001 --no-create-home app

WORKDIR /app
# Root-owned code: the app user can read and run it but not rewrite it.
COPY server.py .

USER 10001
EXPOSE 8080
CMD ["python", "server.py"]
