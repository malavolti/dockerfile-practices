<!--
AUTHORS: Christopher Le
Prefer only GitHub-flavored Markdown in external text.
See README.md for details.
-->

# Good practices on writing Dockerfile

https://chrislevn.github.io/dockerfile-practices/

<!-- markdown="1" is required for GitHub Pages to render the TOC properly. -->

<details markdown="1"> 
  <summary>Table of Contents</summary>
  
- [1 Background](#s1-background)
- [2. Dockerfile's practices](#s2-dockerfile-practices)
    * [2.1 Use minimal base images](#s2.1-minimal-base-image)
    * [2.2 Use explicit tags for the base image.](#s2.2-base-image-explicit-tags)
    * [2.3 Leverage layer caching](#s2.3-leverage-layer-caching)
    * [2.4 Consolidate related operations](#s2.4-consolidate-related-operations)
    * [2.5 Remove unnecessary artifacts](#s2.5-remove-unnecessary-artifacts)
    * [2.6 Use specific COPY instructions](#s2.6-use-specific-copy-instructions)
    * [2.7 Set the correct container user](#s2.7-set-the-correct-container-user)
    * [2.8 Use environment variables for configuration](#s2.8-use-environment-variables-for-configuration)
    * [2.9 Document your Dockerfile](#s2.9-document-your-dockerfile)
    * [2.10 Use .dockerignore file](#s2.10-use-dockerignore)
    * [2.11 Test your image](#s2.11-test-your-image)
- [3 Good demo](#s3-good-demo)
- [4 Contributing guide](#s4-contributing-guide)
- [5 References](#s5-references)
 
</details>

---

## 1 Background

<a id="s1-background"></a>

## 2 Dockerfile's practices
<a id="s2-dockerfile-practices"></a>

### 2.1 Use minimal base images

Start with a minimal base image that contains only the necessary dependencies for your application. Using a smaller image reduces the image size and improves startup time.
```
No: 
  FROM python:3.9
```
```
Yes:
  FROM python:3.9-slim
```
<details>
<summary>Base image types: </summary>
  
  + stretch/buster/jessie: is the codename for Debian 10, which is a specific version of the Debian operating system. Debian-based images often have different releases named after characters from the Toy Story movies. For example, "Jessie" refers to Debian 8, "Stretch" refers to Debian 9, and "Buster" refers to Debian 10. These releases represent different versions of the Debian distribution and come with their own set of package versions and features.
  
  + slim: is a term commonly used to refer to Debian-based base images that have been optimized for size. These images are built on Debian but are trimmed down to include only the essential packages required to run applications. They are a good compromise between size and functionality, providing a balance between a minimal footprint and the availability of a wide range of packages.
  
  + alpine: Alpine Linux is a lightweight Linux distribution designed for security, simplicity, and resource efficiency. Alpine-based images are known for their small size and are highly popular in the Docker ecosystem. They use the musl libc library instead of the more common glibc found in most Linux distributions. Alpine images are often significantly smaller compared to their Debian counterparts but may have a slightly different package ecosystem and may require adjustments to some application configurations due to the differences in the underlying system libraries. However, some teams are moving away from alpine because these images can cause compatibility issues that are hard to debug. Specifically, if using python images, some wheels are built to be compatible with Debian and will need to be recompiled to work with an Apline-based image.

References: 
- https://medium.com/swlh/alpine-slim-stretch-buster-jessie-bullseye-bookworm-what-are-the-differences-in-docker-62171ed4531d
  
</details>

<a id="s2.1-minimal-base-image"></a>

### 2.2 Use explicit tags for the base image.
Use explicit tags for the base image instead of generic ones like 'latest' to ensure the same base image is used consistently across different environments.
 
 ```
No: 
  FROM company/image_name:latest
```
```
Yes:
  FROM company/image_name:version
```

<a id="s2.2-base-image-explicit-tags"></a>

### 2.3 Leverage layer caching
Docker builds images using a layered approach, and it caches each layer. Place the instructions that change less frequently towards the top of the Dockerfile. This allows Docker to reuse cached layers during subsequent builds, speeding up the build process.

```
No:
  FROM ubuntu:20.04

  # Install system dependencies
  RUN apt-get update && \
      apt-get install -y curl

  # Install application dependencies
  RUN curl -sL https://deb.nodesource.com/setup_14.x | bash - && \
      apt-get install -y nodejs

  # Copy application files
  COPY . /app

  # Build the application
  RUN cd /app && \
      npm install && \
      npm run build

```
In this bad example, each `RUN` instruction creates a new layer, making it difficult to leverage layer caching effectively. Even if there are no changes to the application code, every step from installing system dependencies to building the application will be repeated during each build, resulting in slower build times.

```
Yes:
  FROM ubuntu:20.04

  # Install system dependencies
  RUN apt-get update && \
      apt-get install -y curl

  # Install application dependencies
  RUN curl -sL https://deb.nodesource.com/setup_14.x | bash - && \
      apt-get install -y nodejs

  # Set working directory
  WORKDIR /app

  # Copy only package.json and package-lock.json
  COPY package.json package-lock.json /app/

  # Install dependencies
  RUN npm install

  # Copy the rest of the application files
  COPY . /app

  # Build the application
  RUN npm run build
```
In this improved example, we take advantage of layer caching by separating the steps that change less frequently from the steps that change more frequently. Only the necessary files (package.json and package-lock.json) are copied in a separate layer to install the dependencies. This allows Docker to reuse the cached layer for subsequent builds as long as the dependency files remain unchanged. The rest of the application files are copied in a separate step, reducing unnecessary cache invalidation.

<a id="s2.3-leverage-layer-caching"></a>


### 2.4 Consolidate related operations
Minimize the number of layers by combining related operations into a single instruction. For example, instead of installing multiple packages in separate `RUN` instructions, group them together using a single RUN instruction.

```
No:
  FROM ubuntu:20.04

  # Install dependencies
  RUN apt-get update && \
      apt-get install -y curl

  # Install Node.js
  RUN curl -sL https://deb.nodesource.com/setup_14.x | bash - && \
      apt-get install -y nodejs

  # Install project dependencies
  RUN npm install express
  RUN npm install lodash
  RUN npm install axios

```
In this bad example, each package installation is done in a separate `RUN` instruction. This approach creates unnecessary layers and increases the number of cache invalidations. Even if one package changes, all subsequent package installations will be repeated during each build, leading to slower build times.

```
Yes: 
  FROM ubuntu:20.04

  # Install dependencies and Node.js
  RUN apt-get update && \
      apt-get install -y \
          curl \
          nodejs

  # Install project dependencies
  RUN npm install express lodash axios
```
In this improved example, related package installations are consolidated into a single RUN instruction. This approach reduces the number of layers and improves layer caching. If no changes occur in the package.json file, Docker can reuse the previously cached layer for the npm install step, resulting in faster builds.

<a id="s2.4-consolidate-related-operations"></a>

### 2.5 Remove unnecessary artifacts
Clean up any unnecessary artifacts created during the build process to reduce the size of the final image. For example, remove temporary files, unused dependencies, and package caches.

```
No:
  FROM ubuntu:20.04

  # Install dependencies
  RUN apt-get update && \
      apt-get install -y curl

  # Download application package
  RUN curl -O https://example.com/app.tar.gz

  # Extract application package
  RUN tar -xzf app.tar.gz

  # Remove unnecessary artifacts
  RUN rm app.tar.gz
```

In this example, the unnecessary artifacts, such as the downloaded app.tar.gz file, are removed in a separate RUN instruction. However, this approach doesn't take advantage of Docker's layer caching. Even if no changes are made to the downloaded package, Docker will not be able to reuse the cached layer and will repeat the download, extraction, and removal steps during each build.

```
Yes: 
  FROM ubuntu:20.04

  # Install dependencies
  RUN apt-get update && \
      apt-get install -y curl

  # Download and extract application package, then remove unnecessary artifacts
  RUN curl -O https://example.com/app.tar.gz \
      && tar -xzf app.tar.gz \
      && rm app.tar.gz
```

In this improved example, the unnecessary artifacts are removed immediately after they are no longer needed, within the same `RUN` instruction. By doing so, Docker can leverage layer caching effectively. If the downloaded package remains unchanged, Docker can reuse the cached layer, avoiding redundant downloads and extractions.

<a id="s2.5-remove-unnecessary-artifacts"></a>

### 2.6 Use specific COPY instructions
When copying files into the image, be specific about what you're copying. Avoid using . (dot) as the source directory, as it can inadvertently include unwanted files. Instead, explicitly specify the files or directories you need.

```
No: 
  FROM ubuntu:20.04

  # Copy all files into the image
  COPY . /app
```

In this example, the entire context directory, represented by . (dot), is copied into the image. This approach can inadvertently include unwanted files that may not be necessary for the application. It can bloat the image size and potentially expose sensitive files or credentials to the container.

```
Yes:  
  FROM ubuntu:20.04

  # Create app directory
  RUN mkdir /app

  # Copy only necessary files
  COPY app.py requirements.txt /app/

```

In this improved example, specific files (app.py and requirements.txt) are copied into a designated directory (/app). By explicitly specifying the required files, you ensure that only the necessary files are included in the image. This approach helps keep the image size minimal and avoids exposing any unwanted or sensitive files to the container.

<a id="s2.6-use-specific-copy-instructions"></a>

### 2.7 Set the correct container user 
By default, Docker runs containers as the root user. To improve security, create a dedicated user for running your application within the container and switch to that user using the USER instruction.

```
No: 
  FROM ubuntu:20.04

  # Set working directory
  WORKDIR /app

  # Copy application files
  COPY . /app

  # Set container user as root
  USER root

  # Run the application
  CMD ["python3", "app.py"]
```

In this example, the container user is set to root using the USER instruction. Running the container as the root user can pose security risks, as any malicious code or vulnerability exploited within the container would have elevated privileges.

```
Yes: 
  FROM ubuntu:20.04

  # Set working directory
  WORKDIR /app

  # Copy application files
  COPY . /app

  # Create a non-root user
  RUN groupadd -r myuser && useradd -r -g myuser myuser

  # Set ownership and permissions
  RUN chown -R myuser:myuser /app

  # Switch to the non-root user
  USER myuser

  # Run the application
  CMD ["python3", "app.py"]
```

In this improved example, a dedicated non-root user (myuser) is created using the useradd and groupadd commands. The ownership and permissions of the /app directory are changed to the non-root user using chown. Finally, the USER instruction switches to the non-root user before running the application.

<a id="s2.7-set-the-correct-container-user"></a>

### 2.8 Use environment variables for configuration
Instead of hardcoding configuration values inside the Dockerfile, use environment variables. This allows for greater flexibility and easier configuration management. You can set these variables when running the container.

```
No: 
  FROM ubuntu:20.04

  # Set configuration values directly
  ENV DB_HOST=localhost
  ENV DB_PORT=3306
  ENV DB_USER=myuser
  ENV DB_PASSWORD=mypassword

  # Run the application
  CMD ["python3", "app.py"]
```

In this example, configuration values are directly set as environment variables using the ENV instruction in the Dockerfile. This approach has a few drawbacks:

- Configuration values are hardcoded in the Dockerfile, making it less flexible and harder to change without modifying the file itself.
- Sensitive information, such as passwords or API keys, is exposed in plain text in the Dockerfile, which is not secure.

```
Yes: 
  FROM ubuntu:20.04

  # Set default configuration values
  ENV DB_HOST=localhost
  ENV DB_PORT=3306
  ENV DB_USER=defaultuser
  ENV DB_PASSWORD=defaultpassword

  # Run the application
  CMD ["python3", "app.py"]
```

In this improved example, default configuration values are set as environment variables, but they are kept generic and non-sensitive. This approach provides a template for configuration that can be customized when running the container.

To securely provide sensitive configuration values, you can pass them as environment variables during runtime using the -e flag with the docker run command or by using a secrets management solution like Docker Secrets or environment-specific .env files.

For example, when running the container, you can override the default values:

```docker run -e DB_HOST=mydbhost -e DB_PORT=5432 -e DB_USER=myuser -e DB_PASSWORD=mypassword myimage```

<a id="s2.8-use-environment-variables-for-configuration"></a>

### 2.9 Document your Dockerfile
Include comments in your Dockerfile to provide context and explanations for the various instructions. This helps other developers understand the purpose and functionality of the Dockerfile.

```
No: 
  FROM ubuntu:20.04

  # Install dependencies
  RUN apt-get update && apt-get install -y python3

  # Set working directory
  WORKDIR /app

  # Copy application files
  COPY . /app

  # Run the application
  CMD ["python3", "app.py"]
```
In this example, there is no explicit documentation or comments to explain the purpose or functionality of each instruction in the Dockerfile. It can make it challenging for other developers or maintainers to understand the intended usage or any specific requirements.

```
Yes: 
  FROM ubuntu:20.04

  # Install Python
  RUN apt-get update && apt-get install -y python3

  # Set working directory
  WORKDIR /app

  # Copy application files
  COPY . /app

  # Set the entrypoint command to run the application
  CMD ["python3", "app.py"]

  # Expose port 8000 for accessing the application
  EXPOSE 8000

  # Document the purpose of the image and any additional details
  LABEL maintainer="John Doe <john@example.com>"
  LABEL description="Docker image for running the example application."
  LABEL version="1.0"
```

In this improved example, the Dockerfile is better documented:

- Each instruction is accompanied by a comment or description that explains its purpose.
- The `LABEL` instructions are used to provide additional information about the image, such as the maintainer, description, and version.
- The `EXPOSE` instruction documents the port that should be exposed for accessing the application.

<a id="s2.9-document-your-dockerfile"></a>

### 2.10 Use .dockerignore file
The .dockerignore file allow you to exclude files the context like a .gitignore file allow you to exclude files from your git repository.
It helps to make build faster and lighter by excluding from the context big files or repository that are not used in the build.

```
No: 
  FROM ubuntu:20.04

  # Copy all files into the image
  COPY . /app

  # Build the application
  RUN make build
```
In this example, all files in the current directory are copied into the image, including unnecessary files such as development tools, build artifacts, or sensitive information. This can bloat the image size and potentially expose unwanted or sensitive files to the container.

```
Yes: 
  FROM ubuntu:20.04

  # Copy only necessary files into the image
  COPY . /app

  # Build the application
  RUN make build
```

.dockerignore:
```
.git
node_modules
*.log
*.tmp
```

In this improved example, a `.dockerignore` file is used to exclude unnecessary files and directories from being copied into the image. The .dockerignore file specifies patterns of files and directories that should be ignored during the build process. It helps reduce the image size, improve build performance, and avoid including unwanted files.

The `.dockerignore` file in this example excludes the `.git` directory, the `node_modules` directory (common for Node.js projects), and files with extensions `.log` and `.tmp`. These files and directories are typically not needed in the final image and can be safely ignored.

<a id="s2.10-use-dockerignore"></a>

### 2.11 Test your image
After building your Docker image, run it in a container to verify that everything works as expected. This ensures that your image is functional and can be used with confidence.

```
No: 
  FROM ubuntu:20.04

  # Copy application files
  COPY . /app

  # Install dependencies
  RUN apt-get update && apt-get install -y python3

  # Run the application
  CMD ["python3", "app.py"]
```

In this example, there is no explicit provision for testing the image. The Dockerfile only focuses on setting up the application, without any dedicated steps or considerations for running tests.

```
Yes: 
  FROM ubuntu:20.04

  # Copy application files
  COPY . /app

  # Install dependencies
  RUN apt-get update && apt-get install -y python3

  # Run tests
  RUN python3 -m unittest discover tests

  # Run the application
  CMD ["python3", "app.py"]
```

In this improved example, a dedicated step is added to run tests within the Dockerfile. The `RUN` instruction executes the necessary command to run tests using a testing framework (in this case, `unittest` is used as an example). By including this step, you ensure that tests are executed during the Docker image build process.

It's important to note that this example assumes the tests are included in a `tests` directory within the project structure. Adjust the command (`python3 -m unittest discover tests`) as per your project's testing setup.

<a id="s2.11-test-your-image"></a>

## 3 Good demo

<a id="s3-good-demo"></a>

```
# Use a suitable base image
FROM python:3.9-slim-buster

# Set the working directory
WORKDIR /app

# Copy only necessary files
COPY requirements.txt .
COPY app.py .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Set the container user
RUN groupadd -r myuser && useradd -r -g myuser myuser
USER myuser

# Expose the necessary port
EXPOSE 8000

# Set environment variables
ENV DB_HOST=localhost
ENV DB_PORT=3306
ENV DB_USER=defaultuser
ENV DB_PASSWORD=defaultpassword

# Set the entrypoint command
CMD ["python3", "app.py"]
```

In this example, we follow several best practices:

- We use an appropriate base image (`python:3.9-slim-buster`) that provides a minimal Python environment.
- We set the working directory to `/app` to execute commands and copy files within that directory.
- Only necessary files (`requirements.txt` and `app.py`) are copied into the image, reducing unnecessary content.
-Dependencies are installed using pip with the `--no-cache-dir` flag to avoid caching unnecessary artifacts.
-A non-root user (`myuser`) is created and used to run the container, enhancing security.
-The necessary port (`8000`) is exposed to allow access to the application.
-Environment variables are set for configuring the database connection.
-The `CMD` instruction specifies the command to run when the container starts.

---

## 4 Contributing guide

[Contributing guide](https://github.com/chrislevn/dockerfile-practices/blob/main/CONTRIBUTING.md)

<a id="s4-contributing-guide"></a>

## 5 References: 
- https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
- https://medium.com/@adari.girishkumar/dockerfile-and-best-practices-for-writing-dockerfile-diving-into-docker-part-5-5154d81edca4
- https://google.github.io/styleguide/pyguide.html
- https://github.com/dnaprawa/dockerfile-best-practices

<a id="s5-references"></a>


