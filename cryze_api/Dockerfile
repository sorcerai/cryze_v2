# Base image
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

# Set the working directory
WORKDIR /app

# Copy the project file and restore dependencies
COPY *.csproj .
RUN dotnet restore

# Copy the source code and build the application
COPY . .
RUN dotnet publish -c Release -o out

# Runtime image
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

# Install Python 3.11
RUN apt-get update -y && apt-get install -y python3.11 python3-pip ca-certificates

# Wyze's api.wyzecam.com still chains to the legacy "DigiCert Global Root CA",
# which was dropped from 2026+ ca-certificates bundles (only G2/G3/G4/G5 remain).
# Without it, wyze_sdk dies on startup with SSL CERTIFICATE_VERIFY_FAILED
# ("unable to get local issuer certificate"). Re-add the root and point
# Python's requests/ssl at the system bundle.
COPY certs/DigiCertGlobalRootCA.pem /usr/local/share/ca-certificates/DigiCertGlobalRootCA.crt
RUN update-ca-certificates
ENV REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
ENV SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
# make a directory for the database file
RUN mkdir /data
# Set the working directory
WORKDIR /app

# Copy the Python requirements file and install dependencies, do it before copying the published output to leverage Docker's layer caching
COPY ./requirements.txt .
RUN pip install -r /app/requirements.txt --break-system-packages

# Copy the published output from the build stage
COPY --from=build /app/out .

# REQUIRED: Set the environment variable to the port the application will listen on
ENV ASPNETCORE_URLS=http://+:8080
# REQUIRED: set the path to the database file, JsonStoreConfiguration.FilePath
# read by AddEnvironmentVariables() in Program.cs
ENV JsonStoreConfiguration__FilePath=/data/data.json

# Start the application
ENTRYPOINT ["dotnet", "cryze_api.dll"]