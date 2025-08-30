# Laboratorio de Health Checks y Conectividad en Kubernetes

Este laboratorio te guiará a través del despliegue de una aplicación con un delay de inicio, la configuración de health checks para manejar ese delay, y la demostración de la comunicación entre servicios.

## Pre-requisitos
* Cluster de Kubernetes (como minikube, Kind o crc).
* kubectl configurado y autenticado.
* Los archivos de manifiesto necesarios en el directorio `manifests/` (`deployment`, `service`, `ingress`).
* Los archivos de patches en el directorio `patch/`.

---

## Pasos del Laboratorio

### 1. Despliegue de la Aplicación Base

Este comando despliega el **`backend-delay`**, un servicio simple que simula un delay en el inicio. Inicialmente, no tiene configurados los health checks.

```bash
kubectl apply -f manifests/backend-delay
```

Resultado esperado: Se crea un Deployment, Service e Ingress para el backend-delay. La aplicación comenzará a inicializarse, pero el Pod puede quedar en estado Pending o CrashLoopBackOff si tarda demasiado en arrancar.

### 2. Verificar el Estado del Pod

Este comando te permite ver el estado del Pod en el namespace health-checks. Luego de descargar la imágen del contenedor, el estado deseado debe ser **Running**

```bash
kubectl get pod -n health-checks
```


### 3. Modificar el Delay de Inicio
Con este comando, estableces la variable de entorno DELAY_INIT del backend-delay a 60 segundos. Esto generará que la aplicación simule un tiempo de demora en su inicio, que puede ser producto de la propia tecnología o de alguna configuración que genera dicha demora. Esto demostrará la necesidad de un startupProbe adecuado.

```bash
kubectl set env deployment/backend-delay DELAY_INIT=60
```

Resultado esperado: El Deployment se reiniciará con el nuevo valor. El nuevo pod figurará **Running** y **Ready 1/1**.

### 4. Probar la Conectividad
Este comando intenta acceder a la aplicación backend-delay. Dado el delay de 60 segundos, la solicitud de curl probablemente fallará con un timeout o un error de conexión, ya que el Pod si bien esta Running y Ready, el servicio todavía no se encuentra listo para recibir tráfico realmente.

```bash
curl -v http://backend-delay.apps-crc.testing/
```

Resultado esperado: Fallo en la conexión o timeout. La aplicación no está lista para responder.

### 5. Configurar startupProbe

Este comando aplica un patch al Deployment para agregar la startupProbe. Esto le dice a Kubernetes que tiene que esperar el tiempo definido en el parámetro **initialDelaySeconds** para comenzar a realizar otros health checks o, en este caso, para disponibilizar la app. Antes de configurar el health check, vamos a volver a setear un valor más bajo de INITIAL_DELAY

```bash
kubectl set env deployment/backend-delay DELAY_INIT=5
kubectl patch deployment/backend-delay --patch-file=patch/backend-delay-deployment.yaml
```

Resultado esperado: El Deployment se actualizará, y Kubernetes reiniciará el Pod. Con la nueva configuración, el Pod ahora esperará 10 segundos antes de realizar la prueba y, finalmente, pasará al estado Ready.



### 6. Desplegar el Servicio Interconector
Este comando despliega un segundo servicio, service-interconnect, que usaremos para demostrar la comunicación entre servicios dentro del clúster.

```bash
kubectl apply -f manifests/service-interconnect
```

Resultado esperado: Se crea el Deployment, Service e Ingress para el service-interconnect.

### 7. Probar la Conectividad del Nuevo Servicio
Este curl verifica que service-interconnect esté accesible desde fuera del clúster.

```bash
curl http://service-interconnect.apps-crc.testing/
```

Resultado esperado: El servicio responderá con un mensaje de bienvenida.

### 8. Probar la Interconexión de Servicios
Este comando es clave. service-interconnect llamará internamente al backend-delay. La llamada debería ser exitosa, demostrando que los servicios se pueden comunicar dentro del clúster usando sus nombres de servicio.

```bash
curl http://service-interconnect.apps-crc.testing/connect
```
Resultado esperado: Una respuesta del backend-delay que te informará que la conexión entre los servicios fue exitosa.

### 9. Agregar health checks para el servicio interconnect

Estos comandos aplican parches para configurar las sondas de liveness y readiness en el service-interconnect.

```bash
kubectl patch deployment/service-interconnect --patch-file=patch/service-interconnect.yaml
```

Resultado esperado: El Deployment de service-interconnect se actualizará con las nuevas configuraciones, haciéndolo más robusto.

### 10. Pruebas

Cuando se actualiza el despliegue, podemos notar que el service-interconnect nunca pasa a estar Ready. 
Esto se debe a que en el path configurado del health check readiness simulamos una conexión con otro servicio y seteamos un timeout de 2 segundos, siendo que el servicio se demora 4 segundos en responder.

Vamos a configurar la variable de DELAY con un valor más bajo.

```bash
kubectl set env deployment/backend-delay DELAY=1 DELAY_INIT=0
```

Resultado esperado: El Deployment se actualizará con los nuevos valores de delay, y la aplicación service-interconnect estará lista casi de inmediato para comenzar a recibir tráfico.