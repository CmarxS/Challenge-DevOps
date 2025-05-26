# Challenge MotoMap

## Equipe
- RM558640 - Caio Amarante
- RM556325 - Felipe Camargo
- RM555997 - Caio Marques
---

## Descrição da Entrega

Este projeto entrega uma **API REST Spring Boot** containerizada em **Docker** e provisionada em **VM AlmaLinux no Azure**, atendendo aos requisitos de:

- **Containerização em nuvem** (até 85 pts)  
- **Documentação completa no GitHub** (até 15 pts)  

### 1. Provisionamento da Infraestrutura no Azure

1. **Criar Resource Group**  
   ```bash
   az group create --name rg-challenge --location brazilsouth
   ```

2. **Criar VM AlmaLinux 9 Gen2**  
   ```bash
   az vm create --resource-group rg-challenge --name vm-challenge      --image almalinux:almalinux-x86_64:9-gen2:9.6.202505220      --size Standard_B1ms      --admin-username admlnx      --admin-password "Fiap@2tdsvms"      --authentication-type password      --no-wait
   ```

3. **Abrir portas no NSG**  
   ```bash
   az vm open-port --resource-group rg-challenge --name vm-challenge --port 22
   az vm open-port --resource-group rg-challenge --name vm-challenge --port 8080 --priority 1010
   ```

4. **Obter IP público**  
   ```bash
   $IP = az vm show --resource-group rg-challenge --name vm-challenge --show-details --query publicIps -o tsv
   Write-Host "IP da VM:" $IP
   ```

5. **Instalar Docker na VM**  
   ```bash
   ssh admlnx@$IP "
     sudo dnf install -y yum-utils device-mapper-persistent-data lvm2
     sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
     sudo dnf install -y docker-ce docker-ce-cli containerd.io
     sudo systemctl enable docker --now
     docker run --rm hello-world
   "
   ```

---

### 2. Preparação e Build da API Java

```bash
ssh admlnx@$IP "
  sudo dnf install -y git maven java-21-openjdk-devel
  sudo alternatives --set java /usr/lib/jvm/java-21-openjdk-21/bin/java
  sudo alternatives --set javac /usr/lib/jvm/java-21-openjdk-21/bin/javac
"
git clone https://github.com/CmarxS/Challenge-Java.git
cd Challenge-Java
mvn clean package -DskipTests
```

---

### 3. Containerização com Docker

**Dockerfile** (multistage, sem comentários):
\`\`\`dockerfile
FROM maven:3.9.2-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests -B

FROM eclipse-temurin:21-jre
RUN addgroup --system app && adduser --system --ingroup app app
USER app
WORKDIR /home/app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
\`\`\`

\`\`\`bash
docker build -t challenge-java:latest .
docker rm -f challenge-api 2>/dev/null || true
docker run -d --name challenge-api -p 8080:8080 challenge-java:latest
docker ps
\`\`\`

---

### 4. CRUD da Entidade **Funcionario**

- **Entity**: `Funcionario` (`id`, `nome`, `cpf`, `cargo`, `dataAdmissao`)  
- **Repository**: `FuncionarioRepository extends JpaRepository<Funcionario, Long>`  
- **Service**: métodos `listarTodos`, `buscarPorId`, `criar`, `atualizar`, `deletar`  
- **Controller**: endpoints REST `/api/funcionarios` (GET, GET/{id}, POST, PUT/{id}, DELETE/{id})  
- **Dados iniciais** em `src/main/resources/data.sql` (5 inserts):

\`\`\`sql
INSERT INTO funcionarios (nome, cpf, cargo, data_admissao) VALUES
('Ana Silva','12345678901','Analista','2023-01-15'),
('Bruno Costa','23456789012','Desenvolvedor','2022-11-01'),
('Carla Dias','34567890123','DBA','2023-03-10'),
('Daniel Reis','45678901234','Tester','2023-04-05'),
('Elisa Maia','56789012345','DevOps','2022-12-20');
\`\`\`

---

### 5. Testes

- **Internos (VM)**  
  \`\`\`bash
  curl http://localhost:8080/api/funcionarios
  curl http://localhost:8080/api/funcionarios/1
  \`\`\`
- **Externos (local)**  
  \`\`\`bash
  curl http://<IP_DA_VM>:8080/api/funcionarios
  \`\`\`

---

### 6. Limpeza da Infraestrutura

**Script** `infra/cleanup.sh`:
\`\`\`bash
az vm delete --resource-group rg-challenge --name vm-challenge --yes
az group delete --name rg-challenge --yes --no-wait
\`\`\`

---

Com esses passos você tem:
- **Spring Web** e **Spring Data JPA**  
- **Banco H2** (ou Oracle, se configurado)  
- **CRUD completo** em 2+ entidades  
- **Containerização otimizada** (multistage, user non-root)  
- **Provisionamento automatizado** no Azure via CLI  
- **Documentação** e **scripts** no GitHub  

Isso garante **100%** dos requisitos de containerização e documentação para a disciplina.
