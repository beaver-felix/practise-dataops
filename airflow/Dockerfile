FROM apache/airflow:2.7.1-python3.9

# Switch to root to install system packages
USER root

# Install ODBC driver and dependencies
# Remove problematic repos and only install what we need
RUN rm -f /etc/apt/sources.list.d/pgdg.list \
    && apt-get update || true \
    && apt-get install -y --no-install-recommends --no-upgrade \
        unixodbc \
        unixodbc-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Microsoft ODBC Driver
# msodbcsql17 has no arm64 build (Apple Silicon); msodbcsql18 supports amd64 and arm64
RUN mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /etc/apt/keyrings/microsoft.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/microsoft.gpg] https://packages.microsoft.com/debian/11/prod bullseye main" > /etc/apt/sources.list.d/mssql-release.list \
    && apt-get update \
    && ACCEPT_EULA=Y apt-get install -y msodbcsql18 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Docker CLI from official Docker repository
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian bullseye stable" > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y docker-ce-cli \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -f docker \
    && usermod -aG docker airflow

# Switch back to airflow user
USER airflow

# Install Python packages
COPY requirements.txt /opt/airflow/requirements.txt
RUN python -m pip install --upgrade pip \
    && pip install --no-cache-dir -r /opt/airflow/requirements.txt \
    && pip install apache-airflow-providers-docker \
    && pip install dbt-sqlserver \
    && pip check

# Create DBT directory
RUN mkdir -p /opt/airflow/dbt

# Copy DAGs and plugins
COPY --chown=airflow:root dags/ /opt/airflow/dags/
COPY --chown=airflow:root plugins/ /opt/airflow/plugins/

# Copy DBT project - FIXED: Use a volume mount instead of COPY
# The DBT directory will be mounted at runtime via docker-compose.yml 