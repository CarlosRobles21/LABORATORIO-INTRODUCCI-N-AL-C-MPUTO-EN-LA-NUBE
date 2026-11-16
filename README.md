# Laboratorio Serverless con OpenFaaS, Docker y Kubernetes

**Asignatura:** Introducción al Cómputo en la Nube
**Unidad:** Unidad 3 – Competencias Fundamentales en la Nube
**Técnico:** TSU Desarrollo de Software en Código Abierto
**Institución:** Escuela Superior de Innovación y Tecnología (ESIT)
**Tutor:** Vladimir Mate
**Periodo de realización:** 09 al 15 de noviembre
**Duración:** 2 horas

**Estudiante:** _Nombre y Apellido_
**Correo del aula virtual:** _correo@ejemplo.com_

---

## 📌 Descripción

Este repositorio contiene la evidencia del laboratorio de **Computación Serverless y modelo Functions as a Service (FaaS)**.

Se implementa un entorno Serverless local con **OpenFaaS** sobre **Docker Desktop** y **Kubernetes**, y se desarrolla una función en **Python** que:

- Muestra un **formulario web** (nombre, apellido y correo del aula virtual).
- Recibe los datos mediante una solicitud **HTTP POST**.
- Genera un **token UUID único** por registro.
- Obtiene información del **entorno de ejecución** (contenedor Linux).
- Calcula una **firma SHA-256** para verificar la integridad del registro.
- Entrega un **archivo JSON descargable** como resultado.

## 🎯 Objetivo

Implementar un entorno Serverless local mediante OpenFaaS para desarrollar, desplegar y ejecutar una función Python que permita registrar información mediante un formulario web, generar identificadores únicos y producir un archivo JSON descargable como resultado de la ejecución del servicio.

## 🧰 Herramientas utilizadas

| Herramienta | Propósito |
|---|---|
| Docker Desktop | Infraestructura de contenedores |
| Kubernetes (Kubeadm) | Orquestación local |
| Visual Studio Code | Entorno de desarrollo |
| PowerShell | Terminal de comandos |
| Helm | Instalación de OpenFaaS en Kubernetes |
| OpenFaaS CLI (`faas-cli`) | Administración de funciones |
| Python 3 (plantilla `python3-http`) | Lenguaje de la función |

> Se utilizó **OpenFaaS Community Edition (CE)**, orientada al aprendizaje y la experimentación personal.

## 🏗️ Arquitectura

```
Navegador ──► kubectl port-forward (8080) ──► Gateway OpenFaaS
                                                   │
                                                   ▼
                                  Función Python (contenedor Linux)
                                     GET  → formulario HTML
                                     POST → genera JSON descargable
```

Componentes de OpenFaaS en el namespace `openfaas` (gateway, nats, prometheus, alertmanager, queue-worker). Las funciones se despliegan en el namespace `openfaas-fn`.

## 📁 Estructura del proyecto

```
laboratorio-serverless/
├── build/
│   └── registro-serverless/      # Archivos preparados para construir la imagen Docker
│       └── function/
├── registro-serverless/          # Código de la función
│   ├── handler.py
│   ├── handler_test.py
│   ├── requirements.txt
│   └── tox.ini
├── template/                     # Plantillas oficiales de OpenFaaS
│   ├── python3-http/
│   └── ...
├── .gitignore
├── stack.yaml                    # Configuración del despliegue
└── registro_<token>.json         # ✅ Evidencia principal del laboratorio
```

---

## 🚀 Procedimiento

### Parte 1: Instalación de Docker Desktop (Windows 10/11)

**Requisitos mínimos**

| Componente | Requisito |
|---|---|
| Sistema operativo | Windows 11 64 bits (23H2 o posterior) o Windows 10 22H2 |
| Procesador | 64 bits con compatibilidad SLAT |
| RAM | Mínimo 8 GB |
| Virtualización | Habilitada en BIOS/UEFI |
| WSL | Versión 2.1.5 o superior |
| Servicio de Windows | LanmanServer habilitado en inicio automático |

**Verificaciones y WSL 2**

```powershell
winver
wsl --version
wsl --install      # Solo si WSL no está instalado (PowerShell como administrador)
wsl --update
wsl --status
```

**Verificar la instalación de Docker**

```powershell
docker --version
docker info
docker run hello-world
```

Se instaló Docker Desktop **por usuario** con motor **WSL 2**.

---

### Parte 2: Instalación y configuración de OpenFaaS

#### 2.1 Habilitar Kubernetes en Docker Desktop

1. Abrir Docker Desktop → sección **Kubernetes**.
2. Seleccionar **Create cluster**.
3. Elegir el tipo **Kubeadm** (necesario para usar el almacén local de imágenes).
4. Esperar a que el clúster indique que está ejecutándose.

```powershell
kubectl version --client
kubectl config use-context docker-desktop
kubectl get nodes          # El nodo debe estar en estado Ready
```

#### 2.2 Instalar Helm

```powershell
winget install --id Helm.Helm -e
helm version               # Ejecutar en una terminal nueva
```

#### 2.3 Agregar el repositorio de OpenFaaS

```powershell
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm repo update
helm repo list
```

#### 2.4 Crear namespaces e instalar OpenFaaS

```powershell
kubectl create namespace openfaas
kubectl create namespace openfaas-fn

helm upgrade --install openfaas openfaas/openfaas `
  --namespace openfaas `
  --set functionNamespace=openfaas-fn `
  --set generateBasicAuth=true `
  --set serviceType=ClusterIP
```

Verificación:

```powershell
helm list -n openfaas
kubectl get pods -n openfaas
kubectl rollout status -n openfaas deployment/gateway --timeout=300s
kubectl get svc -n openfaas
```

#### 2.5 Instalar OpenFaaS CLI

```powershell
$ErrorActionPreference = "Stop"
$version = (Invoke-RestMethod "https://api.github.com/repos/openfaas/faas-cli/releases/latest").tag_name
$ruta = "$env:USERPROFILE\openfaas-bin"
New-Item -ItemType Directory -Force -Path $ruta | Out-Null
$url = "https://github.com/openfaas/faas-cli/releases/download/$version/faas-cli.exe"
Invoke-WebRequest -Uri $url -OutFile "$ruta\faas-cli.exe"
```

Agregar la carpeta al `PATH` del usuario:

```powershell
$ruta = "$env:USERPROFILE\openfaas-bin"
$pathActual = [Environment]::GetEnvironmentVariable("Path", "User")
if (($pathActual -split ';') -notcontains $ruta) {
    [Environment]::SetEnvironmentVariable("Path", "$pathActual;$ruta", "User")
}
```

Reiniciar la terminal y verificar:

```powershell
faas-cli version
```

#### 2.6 Acceso a la plataforma

Obtener la contraseña de `admin`:

```powershell
$clave = kubectl get secret basic-auth -n openfaas -o jsonpath="{.data.basic-auth-password}"
$password = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($clave))
$password
```

Habilitar el acceso local al Gateway (**mantener esta terminal abierta**):

```powershell
kubectl port-forward -n openfaas svc/gateway 8080:8080
```

En una **segunda terminal**, autenticarse con la CLI:

```powershell
$password | faas-cli login `
  --gateway http://127.0.0.1:8080 `
  --username admin `
  --password-stdin

faas-cli list --gateway http://127.0.0.1:8080
```

La interfaz web queda disponible en `http://127.0.0.1:8080` (usuario: `admin`).

> ⚠️ **Nunca subas la contraseña de OpenFaaS al repositorio.**

---

### Parte 3: Despliegue del servicio Serverless

#### 3.1 Crear la función

Desde la carpeta `laboratorio-serverless`:

```powershell
faas-cli template store pull python3-http
faas-cli new --lang python3-http registro-serverless
```

Esto genera la carpeta `registro-serverless/` y el archivo `stack.yaml`.

#### 3.2 Programar la función

Se reemplazó el contenido de `registro-serverless/handler.py` con el código proporcionado en el aula virtual (`handler.txt`).

| Componente | Función |
|---|---|
| `GET` | Presenta el formulario HTML |
| `POST` | Recibe los datos introducidos |
| `uuid` | Genera un token aleatorio único por registro |
| `platform` | Obtiene características del entorno de ejecución |
| `socket` | Obtiene el nombre del contenedor |
| `datetime` | Registra la fecha y hora |
| `hashlib` | Calcula la firma SHA-256 del contenido |
| `json` | Genera el archivo descargable |

> **Importante:** la función se ejecuta dentro de un contenedor Linux, por lo que los datos del sistema operativo y procesador corresponden al contenedor y no necesariamente a Windows 11.

#### 3.3 Configurar el despliegue (`stack.yaml`)

```yaml
version: 1.0

provider:
  name: openfaas
  gateway: http://127.0.0.1:8080

functions:
  registro-serverless:
    lang: python3-http
    handler: ./registro-serverless
    image: registro-serverless:v1
```

#### 3.4 Usar imágenes locales (una sola vez)

Evita que Kubernetes intente descargar la imagen desde Internet:

```powershell
helm upgrade openfaas openfaas/openfaas `
  --namespace openfaas `
  --reuse-values `
  --set functions.imagePullPolicy=Never
```

#### 3.5 Construir la imagen Docker

Preparar los archivos de construcción:

```powershell
New-Item -ItemType Directory -Force -Path "build\registro-serverless\function" | Out-Null

Copy-Item "template\python3-http\Dockerfile" "build\registro-serverless\"
Copy-Item "template\python3-http\index.py" "build\registro-serverless\"
Copy-Item "template\python3-http\requirements.txt" "build\registro-serverless\"

Copy-Item "registro-serverless\*" "build\registro-serverless\function\" -Recurse -Force
```

Construir y verificar:

```powershell
docker build -t registro-serverless:v1 ./build/registro-serverless
docker images registro-serverless
```

#### 3.5 Ejecutar el servicio

En una terminal de VS Code (mantenerla abierta):

```powershell
docker run --rm -p 8080:8080 registro-serverless:v1
```

Si el puerto 8080 está ocupado por el Gateway de OpenFaaS:

```powershell
docker run --rm -p 8081:8080 registro-serverless:v1
```

Abrir en el navegador `http://localhost:8080` (o `http://localhost:8081` si se usó ese puerto).

#### 3.6 Generar la evidencia

1. Completar el formulario con **nombre, apellido y correo del aula virtual**.
2. Presionar **Generar archivo JSON**.
3. Mover el archivo descargado (`registro_<token>.json`) a la **raíz del proyecto**.

---

## 🧾 Evidencia principal: archivo JSON

El archivo `registro_<token>.json` contiene:

- Nombre, apellido y correo
- Token UUID único
- Fecha y hora (UTC)
- Información del entorno de ejecución
- Firma SHA-256
- Estado de ejecución

Estructura de ejemplo (con datos ficticios):

```json
{
  "nombre": "Nombre",
  "apellido": "Apellido",
  "email": "correo@ejemplo.com",
  "token": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "fecha_utc": "2026-11-06T17:36:34.393364+00:00",
  "sistema_operativo": "Linux",
  "version_so": "6.x.x-microsoft-standard-WSL2",
  "arquitectura": "x86_64",
  "procesador": "",
  "version_python": "3.12.x",
  "hostname_contenedor": "xxxxxxxxxxxx",
  "plataforma": "OpenFaaS",
  "funcion": "registro-serverless",
  "estado": "Ejecucion exitosa",
  "firma_sha256": "..."
}
```

> El archivo JSON real de este laboratorio está en la raíz del repositorio.

---

## 🙈 `.gitignore`

Se creó un `.gitignore` en la raíz que excluye archivos de Python (`__pycache__/`, `*.pyc`, `.tox/`), entornos virtuales, configuración de editores (`.vscode/`, `.idea/`), credenciales (`.env`, `*.pem`, `*.key`, `secrets/`), archivos del sistema, temporales y de Docker (`*.tar`, `*.tar.gz`).

Los archivos `registro_*.json` **no** deben ignorarse, ya que son la evidencia del laboratorio.

---

## 📤 Subir el proyecto a GitHub

```powershell
git init
git add .
git commit -m "Laboratorio Serverless con OpenFaaS"
git branch -M main
git remote add origin https://github.com/<usuario>/<repositorio>.git
git push -u origin main
```

El repositorio debe tener visibilidad **Public** y su enlace se entrega en el espacio designado del aula virtual.

---

## 🛠️ Solución de problemas

| Problema | Posible solución |
|---|---|
| El nodo aparece `NotReady` | Esperar a que Kubernetes termine de iniciar antes de continuar |
| Pods en `ContainerCreating` | Es temporal durante el inicio; esperar y volver a consultar con `kubectl get pods -n openfaas` |
| `faas-cli` no se reconoce | Cerrar y volver a abrir la terminal para actualizar el `PATH` |
| El puerto 8080 está ocupado | Usar `-p 8081:8080` en `docker run` y abrir `http://localhost:8081` |
| La virtualización aparece deshabilitada | Activar Intel VT-x / AMD SVM Mode desde BIOS/UEFI |
| Error de autenticación en `faas-cli` | Volver a obtener la contraseña y repetir `faas-cli login` |
| Kubernetes intenta descargar la imagen | Verificar que se aplicó `functions.imagePullPolicy=Never` y que el clúster es de tipo Kubeadm |

## ✅ Resultados esperados

- [x] Docker Desktop instalado y funcionando
- [x] Clúster de Kubernetes (Kubeadm) activo con nodo `Ready`
- [x] OpenFaaS CE instalado con Helm y accesible desde el navegador
- [x] OpenFaaS CLI configurado y autenticado
- [x] Función Python construida como imagen Docker `registro-serverless:v1`
- [x] Formulario web funcionando
- [x] Archivo JSON generado y almacenado en el repositorio

## 📚 Referencias

- [Documentación de Docker Desktop](https://docs.docker.com/desktop/)
- [Documentación de OpenFaaS](https://docs.openfaas.com/)
- [Helm](https://helm.sh/docs/)
- [Kubernetes](https://kubernetes.io/docs/home/)

---

*Escuela Superior de Innovación y Tecnología (ESIT) – TSU Desarrollo de Software en Código Abierto*
