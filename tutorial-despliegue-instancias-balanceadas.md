### ** Balanceo de Aplicación Web: Implementación Completa: VPC, Instancias, Balanceador y CI/CD con GitHub Actions **

Este tutorial refleja el despliegue de 2 instancias de la Aplicación Web React en subredes privadas y un **Self-Hosted Runner** que se configura en una subred pública. Esto asegura que las instancias web no sean accesibles directamente desde Internet, mejorando la seguridad de la arquitectura.

---

### **1. Contexto del Proyecto**

El objetivo es implementar una arquitectura donde:
1. **Instancias del servicio web**:
   - Se despliegan en subredes privadas y son accesibles únicamente a través del **Application Load Balancer**.
2. **Self-Hosted Runner**:
   - Está en una subred pública para ejecutar Workflows de GitHub Actions que automatizan los despliegues en las instancias privadas.
3. El **Application Load Balancer**:
   - Balancea el tráfico HTTP entre las instancias privadas en diferentes zonas de disponibilidad.

---

### **2. Configuración de la VPC**

#### **Paso 1. Crear la VPC**
1. Ve a **AWS > VPC > Your VPCs > Create VPC**.
2. Configura:
   - **Nombre**: `tic-tac-toe-vpc`.
   - **Rango CIDR IPv4**: `10.0.0.0/16`.
3. Haz clic en **Create VPC**.

---

#### **Paso 2. Crear Subredes**
1. **Subredes Públicas**:
   - **Subnet 1**:
     - Nombre: `tic-tac-toe-public-1`.
     - Zona de disponibilidad: `us-east-1a`.
     - CIDR: `10.0.1.0/24`.
   - **Subnet 2**:
     - Nombre: `tic-tac-toe-public-2`.
     - Zona de disponibilidad: `us-east-1b`.
     - CIDR: `10.0.2.0/24`.

2. **Subredes Privadas**:
   - **Subnet 3**:
     - Nombre: `tic-tac-toe-private-1`.
     - Zona de disponibilidad: `us-east-1a`.
     - CIDR: `10.0.3.0/24`.
   - **Subnet 4**:
     - Nombre: `tic-tac-toe-private-2`.
     - Zona de disponibilidad: `us-east-1b`.
     - CIDR: `10.0.4.0/24`.

---

#### **Paso 3. Configuración del Internet Gateway y las Tablas de Rutas**
1. **Internet Gateway**:
   - Si no se creó automáticamente, ve a **VPC > Internet Gateways > Create Internet Gateway**.
   - Asócialo a la VPC.

2. **Tabla de Rutas para Subredes Públicas**:
   - Asegúrate de que la tabla de rutas asociada a las subredes públicas tenga una ruta que apunte a `0.0.0.0/0` y esté dirigida al Internet Gateway.

3. **Tabla de Rutas para Subredes Privadas**:
   - Verifica que las subredes privadas usen la tabla de rutas predeterminada, que solo permite tráfico interno dentro de la VPC.

---

### **3. Aprovisionamiento de Instancias EC2**

#### **Paso 1. Crear Instancias para el Servicio Web**
1. **Instancia 1**:
   - Subnet: `tic-tac-toe-private-1`.
2. **Instancia 2**:
   - Subnet: `tic-tac-toe-private-2`.

#### **Paso 2. Crear la Instancia para el Self-Hosted Runner**
1. Crea una instancia en:
   - Subnet: `tic-tac-toe-public-1`.

---


### **4. Configuración del Application Load Balancer**

1. **Configura el Balanceador**:
   - Coloca el **Load Balancer** en las subredes públicas.
   - Asocia el Target Group a las instancias en las subredes privadas.
2. **Health Checks**:
   - Asegúrate de que `/index.html` sea el endpoint configurado para los Health Checks.

---


### **5. Configuración del Self-Hosted Runner**

#### **Paso 1. Instalar Docker**
El **Self-Hosted Runner** requiere Docker para ejecutar acciones de GitHub basadas en contenedores.

1. Conéctate a la instancia del Runner a través EC2 Instance Connect   

2. Instala Docker:
   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io
   sudo usermod -aG docker ubuntu
   ```

3. Habilita y verifica Docker:
   ```bash
   sudo systemctl enable docker
   sudo systemctl start docker
   docker --version
   ```

#### **Paso 2. Asociar el Runner a GitHub**
1. Ve a **GitHub > Settings > Actions > Runners > Add New Runner**.
2. Sigue las instrucciones proporcionadas para registrar el Runner. Esto incluye:
   - Descargar el binario del Runner.
   - Configurarlo con el token proporcionado por GitHub.

3. Instala el Runner como un servicio para que se ejecute en segundo plano:
   ```bash
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```
---

### **6. Configuración del Workflow**

El Workflow YAML debe incluir un `matrix` en la estrategia de despliegue para enviar artefactos a múltiples instancias.

#### **Workflow YAML**
```yaml
name: CI/CD Despliegue React a EC2

on:
  push:
    branches:
      - main

jobs:
  build:
    name: Compilar React App
    runs-on: ubuntu-latest

    steps:
      - name: Checkout del repositorio
        uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Instalar dependencias
        run: npm install

      - name: Compilar aplicación
        run: npm run build

      - name: Subir artefacto
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: dist

  deploy:
    name: Desplegar en EC2
    runs-on: ub-tictactoe-runner
    needs: build

    strategy:
      matrix:
        ec2_instance: [ "EC2_HOST_1", "EC2_HOST_2" ]

    steps:
      - name: Descargar artefacto
        uses: actions/download-artifact@v4
        with:
          name: build-artifact
          path: .

      - name: Copiar archivos al servidor EC2
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets[matrix.ec2_instance] }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "./*"
          target: "/var/www/tic-tac-toe"

      - name: Reiniciar Nginx
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets[matrix.ec2_instance] }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            sudo systemctl restart nginx
```

---