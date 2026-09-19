# Docker Image Layer Optimization

## Image Layer Optimization in a Startup

### Scenario

A startup notices that its Docker images are large and image builds are taking too much time. This practical demonstrates how a better Dockerfile strategy can reduce image size and improve rebuild performance.

The experiment compares a poorly optimized Dockerfile with an optimized Dockerfile by using:

* Docker image layers
* Docker layer caching
* Smaller base images
* `.dockerignore`
* Dependency-first copying
* `docker history`
* Docker build-time measurement

---

## Aim

To understand Docker image layers and demonstrate how proper Dockerfile instruction ordering can reduce unnecessary rebuild work and improve Docker image efficiency.

---

## Learning Objectives

By completing this practical, the following concepts are demonstrated:

* Creating a Dockerfile
* Understanding Docker image layers
* Understanding `FROM`, `WORKDIR`, `COPY`, `RUN`, and `CMD`
* Understanding Docker layer caching
* Comparing a bad Dockerfile with an optimized Dockerfile
* Using `docker build`
* Using `docker run`
* Using `docker history`
* Measuring Docker build time
* Understanding the importance of `.dockerignore`

---

## Key Concept

Docker images are built as a sequence of layers.

A good Dockerfile places files that change less frequently earlier in the Dockerfile and files that change frequently later.

```text
Rarely changing files
        ↓
Dependency installation
        ↓
Frequently changing application code
```

This allows Docker to reuse cached layers when possible.

---

## Project Structure

```text
optimizedapp/
│
├── app.py
├── requirements.txt
├── Dockerfile.bad
├── Dockerfile
├── .dockerignore
├── README.md
└── screenshots/
```

---

## Application

The application is intentionally simple because the main focus of this practical is Docker image optimization.

### `app.py`

```python
print("Hello from optimized Docker app")
```

---

## Dependencies

The `requirements.txt` file contains:

```text
flask==3.0.0
requests==2.31.0
```

These dependencies are used to demonstrate why dependency installation should be placed in a reusable Docker layer.

---

# Bad Dockerfile

The first Dockerfile intentionally uses an inefficient layer order.

### `Dockerfile.bad`

```dockerfile
FROM python:3.11

WORKDIR /app

COPY . .

RUN pip install -r requirements.txt

CMD ["python", "app.py"]
```

### Problem with this approach

The complete project is copied before dependencies are installed.

Therefore, when application code changes:

```text
COPY . .
    ↓
Layer changes
    ↓
Dependency installation layer becomes invalid
    ↓
pip install runs again
```

This can make repeated builds unnecessarily expensive.

---

# Optimized Dockerfile

The optimized Dockerfile uses a smaller base image and a better layer order.

### `Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]
```

### Why it is optimized

The dependency file is copied separately before the application source code.

```text
requirements.txt
        ↓
pip install
        ↓
app.py
```

If only `app.py` changes, Docker can reuse the dependency installation layer as long as `requirements.txt` remains unchanged.

---

# `.dockerignore`

The project also uses `.dockerignore` to prevent unnecessary files from being included in the Docker build context.

```text
__pycache__
*.pyc
*.pyo
*.pyd
.git
.gitignore
Dockerfile.bad
README.md
```

This helps keep the build context clean and prevents unnecessary files from being sent to Docker.

---

# Docker Image Comparison

## Build the Bad Image

```powershell
docker build -f Dockerfile.bad -t optimizedapp:bad .
```

## Build the Optimized Image

```powershell
docker build -t optimizedapp:v1 .
```

---

# Running the Containers

### Bad Image

```powershell
docker run --rm optimizedapp:bad
```

### Optimized Image

```powershell
docker run --rm optimizedapp:v1
```

Expected output:

```text
Hello from optimized Docker app
```

The `--rm` option automatically removes the container after it stops.

---

# Inspecting Docker Image Layers

Docker image layers can be inspected using:

```powershell
docker history optimizedapp:bad
```

and:

```powershell
docker history optimizedapp:v1
```

This allows the image layers created by the Dockerfile instructions to be examined.

Important points to observe include:

* Base image layers
* Dependency installation layer
* Application copy layer
* Differences between `python:3.11` and `python:3.11-slim`

---

# Measuring Build Time

Since this practical is performed on Windows PowerShell, build time can be measured using `Measure-Command`.

### Bad Dockerfile

```powershell
Measure-Command { docker build -f Dockerfile.bad -t optimizedapp:bad . }
```

### Optimized Dockerfile

```powershell
Measure-Command { docker build -t optimizedapp:v1 . }
```

The `real` equivalent for PowerShell is the total elapsed time represented by the `Measure-Command` output.

---

# Rebuild Demonstration

The most important part of the experiment is demonstrating what happens when only application code changes.

The application can be modified to:

```python
print("Hello from optimized Docker app - updated version")
```

The images can then be rebuilt.

### Bad Dockerfile Rebuild

```powershell
Measure-Command { docker build -f Dockerfile.bad -t optimizedapp:bad2 . }
```

### Optimized Dockerfile Rebuild

```powershell
Measure-Command { docker build -t optimizedapp:v2 . }
```

With the optimized Dockerfile, the dependency layer can remain cached when only `app.py` changes.

---

# Observation

| Scenario                            | Dockerfile Strategy                       | Expected Behavior                                |
| ----------------------------------- | ----------------------------------------- | ------------------------------------------------ |
| Initial bad build                   | Full source copied before dependencies    | Larger base image and inefficient layer ordering |
| Initial optimized build             | Slim base + dependency-first ordering     | Smaller and better structured image              |
| Bad rebuild after code change       | `COPY . .` before installation            | Dependency installation may run again            |
| Optimized rebuild after code change | Dependencies installed before source copy | Dependency layer can be reused from cache        |

---

# Important Docker Commands

| Purpose                     | Command                                                                    |
| --------------------------- | -------------------------------------------------------------------------- |
| Build bad image             | `docker build -f Dockerfile.bad -t optimizedapp:bad .`                     |
| Build optimized image       | `docker build -t optimizedapp:v1 .`                                        |
| Run bad image               | `docker run --rm optimizedapp:bad`                                         |
| Run optimized image         | `docker run --rm optimizedapp:v1`                                          |
| View bad image layers       | `docker history optimizedapp:bad`                                          |
| View optimized image layers | `docker history optimizedapp:v1`                                           |
| Measure bad build           | `Measure-Command { docker build -f Dockerfile.bad -t optimizedapp:bad . }` |
| Measure optimized build     | `Measure-Command { docker build -t optimizedapp:v1 . }`                    |

---

# Screenshots

The `screenshots` folder contains evidence of the practical execution, including:

1. Project structure
2. `app.py`
3. `requirements.txt`
4. `Dockerfile.bad`
5. Optimized `Dockerfile`
6. `.dockerignore`
7. Bad Docker image build
8. Optimized Docker image build
9. Running application
10. `docker history` output
11. Build-time measurement
12. Rebuild demonstrating Docker cache

---

# Result

The practical demonstrated that Dockerfile design directly affects image efficiency and rebuild behavior.

The optimized approach uses:

```text
python:3.11-slim
        ↓
COPY requirements.txt
        ↓
Install dependencies
        ↓
COPY app.py
```

This allows Docker to reuse the dependency installation layer when only the application source code changes.

---

# Key Learning

The main lesson from this experiment is:

> Copy rarely changing files first and frequently changing files later.

Proper Docker layer ordering helps Docker reuse cached layers and avoid repeating expensive operations unnecessarily.

Using a smaller base image and `.dockerignore` also helps reduce unnecessary image and build-context contents.

---

## Conclusion

Docker image optimization is not only about reducing image size. Proper Dockerfile structure also improves build efficiency by making better use of Docker's layer cache.

The experiment demonstrated this concept through Docker image builds, container execution, `docker history`, and build-time measurements.
