# Lab 2 — Kubernetes local: Deployment, Service e Ingress (kind)

## Objetivo
Desplegar la imagen endurecida del Lab 1 en un cluster Kubernetes local, construyendo la cadena completa de exposición: Deployment → Service → Ingress, con toda la infraestructura declarada en YAML y versionada en git.

## Arquitectura final
```
curl localhost:80
   → puerta Docker (extraPortMappings 80/443)
     → nginx Ingress Controller
       → Ingress hello-ingress (regla: / → hello-svc)
         → Service hello-svc (ClusterIP, 80 → 8080)
           → 2 pods (Deployment hello, imagen hello-good:latest)
             → Flask responde "Hello DevSecOps"
```

## Componentes (escritos a mano, en este repo)
- **kind-config.yaml** — cluster con `extraPortMappings` (80/443): kind corre el nodo como contenedor Docker; sin publicar esos puertos, el tráfico HTTP muere en la puerta
- **deployment.yaml** — 2 réplicas, `imagePullPolicy: Never` (imagen local cargada con `kind load docker-image`, porque el nodo usa containerd y no ve el daemon Docker del host)
- **service.yaml** — ClusterIP `hello-svc`, selector por label `app: hello`, puerto 80 → targetPort 8080
- **ingress.yaml** — regla path `/` (Prefix) → backend `hello-svc:80`, materializada por el nginx Ingress Controller

## Comandos clave
```bash
kind create cluster --name lab --config kind-config.yaml
kind load docker-image hello-good:latest --name lab
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl apply -f deployment.yaml -f service.yaml -f ingress.yaml
curl localhost   # → Hello DevSecOps
```

## Incidentes encontrados y diagnosticados (post-mortem)
1. **La carrera del curl** — curl ejecutado antes de que el port-forward levantara → *verifica que el fallo existe antes de diagnosticarlo*
2. **Puerto ocupado** — un port-forward viejo respondió por el nuevo: resultado verde por el camino equivocado → *verifica por dónde pasa el tráfico; falsa confianza es peor que un error*
3. **Imagen fantasma** — el nodo (containerd) no ve las imágenes del Docker local → `kind load` (en EKS: push a ECR + pull del nodo)
4. **La puerta inexistente** — todo sano dentro del cluster pero `connection refused` desde el host; `docker port lab-control-plane` solo mostraba 6443 → faltaban los extraPortMappings → *cuando todo adentro está sano, el problema está en la frontera*

## Disaster recovery declarativo
El incidente 4 se resolvió **recreando el cluster completo desde los manifiestos en git en ~10 minutos**: delete cluster → create con config correcta → load image → controller → apply de los 3 YAML. Nada vive solo en el cluster; git es la fuente de verdad.

## Lecciones
1. **Service = identidad estable sobre pods efímeros** — el diagnóstico #1 de un service que no responde: `kubectl get endpoints` (vacío = selector no matchea)
2. **Ingress declara reglas; el Controller las materializa** — nginx en local, AWS Load Balancer Controller (→ ALB real) en EKS. Mismo YAML conceptual, distinto motor: portabilidad
3. **port vs targetPort** — el contrato público del Service desacoplado del puerto interno del contenedor
4. **Peculiaridades de kind ≠ Kubernetes universal** — extraPortMappings y kind load son folklore de laboratorio; Deployment/Service/Ingress corren idénticos en cualquier cloud