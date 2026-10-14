# 🚀 Pipeline CI/CD — Spring Boot con Jenkins, SonarQube y Artifactory

Guía paso a paso para desplegar Jenkins en Docker Desktop y ejecutar el pipeline completo con los stages: **Build → Testing (JUnit + JaCoCo) → Sonar → Artifactory**.

---

## 📋 Prerrequisitos

| Herramienta | Versión mínima | Verificar |
|---|---|---|
| Docker Desktop | 4.x | `docker --version` |
| Docker Compose (incluido en Docker Desktop) | 2.x | `docker compose version` |
| Git | cualquiera | `git --version` |
| Puertos libres | 8080, 8082, 9000, 9001, 50000 | — |

---

## 1. Crear la red Docker compartida

Todos los contenedores se comunicarán en la misma red.

```bash
docker network create devops-net
```

---

## 2. Instalar Jenkins en Docker

### 2.1 Levantar el contenedor Jenkins

```bash
docker run -d 
  --name jenkins 
  --network devops-net 
  -p 8080:8080 
  -p 50000:50000 
  -v jenkins_home:/var/jenkins_home 
  -v /var/run/docker.sock:/var/run/docker.sock 
  jenkins/jenkins:lts-jdk21
```

> **Nota:** `-v /var/run/docker.sock` permite que Jenkins ejecute comandos Docker desde dentro del contenedor (opcional para este ejercicio).

### 2.2 Obtener la contraseña inicial de administrador

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```


### 2.3 Completar el wizard de Jenkins

1. Abre el navegador: `http://localhost:8080`
2. Pega la contraseña inicial.
3. Selecciona **"Install suggested plugins"** y espera que termine.
4. Crea el usuario administrador (anota usuario y contraseña).
5. Confirma la URL de Jenkins (`http://localhost:8080`) → **Save and Finish**.

---

## 3. Instalar los plugins necesarios

Ve a **Manage Jenkins → Plugins → Available plugins** y busca e instala:

| Plugin | Descripción |
|---|---|
| **Pipeline** | (ya incluido en "suggested plugins") |
| **Pipeline: Stage View** | Vista gráfica de stages |
| **Git** | Integración con repositorios Git |
| **Gradle** | Soporte para builds con Gradle |
| **JaCoCo** | Publicar reportes de cobertura |
| **JUnit** | Publicar resultados de tests (ya incluido) |
| **SonarQube Scanner** | Integración con SonarQube |
| **Artifactory** | Subir artefactos a JFrog Artifactory |
| **Workspace Cleanup** | Limpia workspace (`cleanWs()`) |

```
Manage Jenkins → Plugins → Available plugins
→ Buscar e instalar cada plugin de la tabla anterior de manera individual
→ Marcar "Restart Jenkins when installation is complete"
```

---

## 4. Instalar SonarQube en Docker

```bash
docker run -d \
  --name sonarqube \
  --network devops-net \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:community
```

### 4.1 Primer acceso a SonarQube

1. Abre: `http://localhost:9000`
2. Login: `admin` / `admin` (cambia la contraseña cuando lo pida).

### 4.2 Generar token de autenticación

```
My Account (ícono superior derecho) → Security → Generate Tokens
→ Name: jenkins-token
→ Type: Global Analysis Token
→ Generate → token generado
```

### 4.3 Crear proyecto en SonarQube

```
Projects → Create Project → Manually
→ Project key:  spring-boot-complete
→ Display name: Spring Boot Complete
→ Set Up → Locally → Use existing token (pega el token) !!!
```

---

## 5. Instalar Artifactory OSS en Docker

```bash
cd .\artifactory\
```
```bash
docker compose up -d
```

### 5.1 Primer acceso a Artifactory

1. Abre: `http://localhost:8082`
2. Login: `admin` / `password` → cambia la contraseña. 
3. Completa el wizard de configuración inicial.

### 5.2 Crear repositorio local

```
Administration → Repositories → Add Repositories → Local Repository
→ Package Type: Gradle (o Maven/Generic)
→ Repository Key: libs-release-local
→ Save
```

### 5.3 Generar API Key o Identity Token

```
Edit Profile (esquina superior derecha) → Authentication Settings
→ Generate an API Key 
```

```
Generate identity token
Usernameadmin
Scopeapplied-permissions/user
Audience*@*
Expires In 11 month(s) 30 day(s)
Token IDe3f497f4-5bab-4553-8570-c133acd67c69
Reference Token

cmVmdGtuOjAxOjE4MjM1NDAyNTY6WDg3Sm0wQjdjVDlFYWdEMWNjVW1JTGp3UU5h
```

---

## 6. Configurar Jenkins — Global Tool Configuration

Ve a **Manage Jenkins → Tools**.

### 6.1 JDK 21

```
JDK → Add JDK
→ Name: JDK-21
→ Desmarca "Install automatically"
→ JAVA_HOME: /opt/java/openjdk   (ruta dentro del contenedor Jenkins LTS JDK21)
```

> Para verificar la ruta exacta:
> ```bash
> docker exec jenkins java -XshowSettings:all -version 2>&1 | grep java.home
> ```

### 6.2 Gradle 8

```
Gradle → Add Gradle
→ Name: Gradle-8
→ Marca "Install automatically"
→ Version: Gradle 8.x (la más reciente disponible)
→ Save
```

---

## 7. Configurar credenciales en Jenkins

Ve a **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**.

### 7.1 Token de SonarQube

```
Kind: Secret text
Secret: <token generado en SonarQube>
ID: sonarqube-token
Description: SonarQube Analysis Token
```

### 7.2 Credenciales de Artifactory

```
Kind: Username with password
Username: admin
Password: <contraseña de Artifactory>
ID: artifactory-credentials
Description: Artifactory Admin Credentials
```

---

## 8. Configurar SonarQube en Jenkins

Ve a **Manage Jenkins → Configure System → SonarQube servers**.

```
☑ Environment variables
Add SonarQube:
→ Name:           SonarQube          ← debe coincidir con withSonarQubeEnv('SonarQube')
→ Server URL:     http://sonarqube:9000
→ Server auth token: sonarqube-token (credencial creada en el paso 7.1)
→ Save
```

---

## 9. Configurar Artifactory en Jenkins

Ve a **Manage Jenkins → Configure System → JFrog**.

```
Add JFrog Platform Instance:
→ Instance ID: Artifactory          ← debe coincidir con serverId: 'Artifactory'
→ JFrog Platform URL: http://artifactory:8082
→ Credentials: artifactory-credentials
→ Test Connection → Save
```

---

## 10. Configurar el webhook de SonarQube (Quality Gate)

El stage `waitForQualityGate` requiere que SonarQube notifique a Jenkins.

En SonarQube: **Administration → Configuration → Webhooks → Create**

```
Name: jenkins
URL:  http://jenkins:8080/sonarqube-webhook/
→ Create
```

---

## 11. Verificar que el plugin de SonarQube en build.gradle

El Jenkinsfile usa `./gradlew sonar`. Para eso, `build.gradle` debe tener:

```groovy
plugins {
    id 'org.sonarqube' version '5.1.0.4882'
}
```

Agrega esta línea al bloque `plugins` del archivo [`build.gradle`](build.gradle):

```bash
# Edita build.gradle y agrega en el bloque plugins:
id 'org.sonarqube' version '5.1.0.4882'
```

> Verifica la versión más reciente en: https://plugins.gradle.org/plugin/org.sonarqube

---

## 12. Crear el pipeline en Jenkins

### 12.1 Nuevo Item

```
Jenkins Dashboard → New Item
→ Name: spring-boot-pipeline
→ Type: Pipeline
→ OK
```

### 12.2 Configurar el pipeline

En la sección **Pipeline**:

**Opción A — Archivo Jenkinsfile desde SCM (recomendado)**

```
Definition: Pipeline script from SCM
SCM: Git
Repository URL: <URL de tu repositorio Git>
Branch: */main (o */master)
Script Path: Jenkinsfile
→ Save
```

**Opción B — Pegar el Jenkinsfile directamente**

```
Definition: Pipeline script
→ Pega el contenido del archivo Jenkinsfile
→ Save
```

---

## 13. Ejecutar el pipeline

```
spring-boot-pipeline → Build Now
```

Monitorea el avance en:
- **Stage View**: vista gráfica de los stages.
- **Console Output**: logs detallados de cada stage.
- **SonarQube** (`http://localhost:9000`): resultado del análisis y Quality Gate.
- **Artifactory** (`http://localhost:8082`): artefacto publicado en `libs-release-local`.

---

## 14. Descripción de los stages del Jenkinsfile

| Stage | Comando Gradle | Salida |
|---|---|---|
| **Build** | `./gradlew clean assemble` | Clases compiladas en `build/` |
| **Testing** | `./gradlew test jacocoTestReport` | Reportes JUnit XML + JaCoCo HTML/XML |
| **Sonar** | `./gradlew sonar` | Análisis publicado en SonarQube, Quality Gate evaluado |
| **Artifact** | `./gradlew bootJar` + `rtUpload` | JAR ejecutable subido a Artifactory |

### Rutas de reportes generados

```
build/test-results/test/*.xml          ← JUnit (consumido por Jenkins)
build/reports/tests/test/index.html    ← JUnit HTML
build/reports/jacoco/test/jacocoTestReport.xml   ← JaCoCo XML (SonarQube)
build/reports/jacoco/test/html/index.html        ← JaCoCo HTML (Jenkins)
build/libs/spring-boot-complete-*.jar            ← artefacto final
```

---

## 15. Comandos de diagnóstico útiles

```bash
# Ver logs de Jenkins
docker logs -f jenkins

# Ver logs de SonarQube
docker logs -f sonarqube

# Ver logs de Artifactory
docker logs -f artifactory

# Reiniciar Jenkins
docker restart jenkins

# Acceder al shell de Jenkins
docker exec -it jenkins bash

# Verificar conectividad entre contenedores (desde Jenkins hacia SonarQube)
docker exec jenkins curl -s http://sonarqube:9000/api/system/status

# Verificar conectividad hacia Artifactory
docker exec jenkins curl -s http://artifactory:8082/artifactory/api/system/ping
```

---

## 16. Troubleshooting

| Problema | Solución |
|---|---|
| `./gradlew: Permission denied` | Agrega `sh 'chmod +x gradlew'` antes del primer `sh` |
| Quality Gate timeout | Verifica que el webhook esté configurado en SonarQube (paso 10) |
| `sonar` task not found | Agrega el plugin `org.sonarqube` en `build.gradle` (paso 11) |
| Artifactory connection refused | Verifica que ambos contenedores estén en `devops-net` |
| JaCoCo exec not found | Confirma que `test.finalizedBy jacocoTestReport` está en `build.gradle` |
| Jenkins no arranca | `docker logs jenkins` — puede ser falta de memoria en Docker Desktop (asigna ≥4 GB RAM) |

---

## 17. Detener y limpiar el entorno

```bash
# Detener contenedores
docker stop jenkins sonarqube artifactory

# Eliminar contenedores (los volúmenes persisten)
docker rm jenkins sonarqube artifactory

# Eliminar volúmenes (CUIDADO: elimina toda la configuración)
docker volume rm jenkins_home artifactory_data

# Eliminar la red
docker network rm devops-net
```

---

## Resumen de URLs

| Servicio | URL |
|---|---|
| Jenkins | http://localhost:8080 |
| SonarQube | http://localhost:9000 |
| Artifactory | http://localhost:8082 |

---


