## Prompt
```
[BEFORE] 
Aqui está um manifest de Deployment do Kubernetes legado (`chronos-api`) de um ambiente de produção. Ele apresenta vários antipadrões graves: usa a tag de imagem `latest`, possui apenas 1 réplica configurada (ponto único de falha), expõe credenciais sensíveis (DB_PASSWORD e JWT_SECRET) em texto plano no array `env`, e não possui configurações básicas de resiliência e segurança, como limits/requests, health checks e security context.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"

[AFTER] 
Preciso da versão modernizada deste manifest, alinhada com as melhores práticas para ambientes enterprise. O resultado deve ter alta disponibilidade (mínimo de 3 réplicas), utilizar uma imagem versionada (ex: `v2.1.0`), carregar as secrets de forma segura usando `secretKeyRef` (assumindo um Secret chamado `chronos-api-secrets`), definir `resources` (requests e limits dimensionados para uma API padrão), incluir `livenessProbe` e `readinessProbe` baseados na porta 8080, e aplicar um `securityContext` rigoroso no container (não-root, sistema de arquivos read-only, e sem escalação de privilégios).

[BRIDGE] 
Reescreva o código YAML completo aplicando todas essas exigências. Inclua comentários curtos e diretos no próprio código explicando cada nova adição de segurança e resiliência. Não gere explicações fora do bloco de código YAML, entregue apenas o manifest pronto para ser aplicado.
```
---
## Modelo
```
Claude Opus 4.7 Adaptativo
```
---
## Output
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
    version: v2.1.0
spec:
  # Alta disponibilidade: 3 réplicas eliminam o SPOF e permitem rolling updates sem downtime
  replicas: 3
  # Estratégia de rollout controlada para garantir disponibilidade durante deploys
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
        version: v2.1.0
    spec:
      # SecurityContext no Pod: define UID/GID não-root e fsGroup para volumes
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      # automountServiceAccountToken desabilitado caso a aplicação não consuma a API do K8s
      automountServiceAccountToken: false
      containers:
        - name: api
          # Imagem versionada (immutable tag) — garante reprodutibilidade e rollback confiável
          image: chronos-api:v2.1.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          # Secrets carregadas via secretKeyRef — nunca em texto plano no manifest
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: chronos-api-secrets
                  key: db-password
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: chronos-api-secrets
                  key: jwt-secret
          # Requests/Limits: requests garantem scheduling previsível; limits evitam noisy neighbors
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          # Liveness: reinicia o container caso a aplicação trave (deadlock, loop infinito)
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3
          # Readiness: remove o pod do Service até estar pronto para receber tráfego
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          # Startup probe: protege apps com cold start lento de ser morto pelo liveness
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30
          # SecurityContext do container: hardening rigoroso conforme Pod Security Standards (restricted)
          securityContext:
            runAsNonRoot: true
            runAsUser: 10001
            runAsGroup: 10001
            # Sistema de arquivos imutável — bloqueia escrita inesperada e mitiga tampering em runtime
            readOnlyRootFilesystem: true
            # Bloqueia ganho de privilégios via setuid/setgid (ex: sudo, su)
            allowPrivilegeEscalation: false
            privileged: false
            # Drop ALL capabilities: princípio do menor privilégio no nível do kernel
            capabilities:
              drop:
                - ALL
          # Volumes ephemeral para diretórios que a app precisa escrever (compatível com readOnlyRootFilesystem)
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /var/cache/app
      volumes:
        - name: tmp
          emptyDir:
            medium: Memory
            sizeLimit: 64Mi
        - name: cache
          emptyDir:
            sizeLimit: 128Mi
      # Spread entre nodes para resiliência a falha de nó/zona
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: chronos-api
      terminationGracePeriodSeconds: 30
```
---
## Justificativa
```
O framework BAB foi ideal para este caso porque espelha o fluxo natural de uma refatoração em SRE/DevOps: o Before contextualizou o manifest legado e seus antipadrões, o After definiu critérios objetivos e verificáveis para o estado desejado (réplicas, secretKeyRef, probes, securityContext), e o Bridge delimitou o formato da entrega de forma acionável. Essa estrutura reduz ambiguidade e replica o modelo mental de um gap analysis ou remediação de compliance (CIS Benchmarks, Pod Security Standards), gerando outputs mais precisos do que prompts narrativos abertos.
```