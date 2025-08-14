Passo a passo para criação e configuração da infraestrutura.

# Pré-requisitos
Alguns serviços utilizados dependem da criação prévia de security groups com regras de entra e saída corretamente estipuladas. Para acessar o passo da criação desses recursos clique aqui.

# VPC (VIrtual Private Cloud)

### Criação da VPC
No console da AWS vá em VPC > Your VPCs > Create VPC
![VPC SETTINGS](Images/VPC/vpc-settings.PNG)

### Criação das Subnets
No console da AWS vá em VPC > Subnets > Create subnets

1. Subnet public-az1
![Subnet public-az1](Images/VPC/Subnets/public-az1.PNG)

2. Subnet public-az2
![Subnet public-az2](Images/VPC/Subnets/public-az2.PNG)

3. Subnet data-az1
![Subnet data-az1](Images/VPC/Subnets/data-az1.PNG)

4. Subnet data-az2
![Subnet data-az2](Images/VPC/Subnets/data-az2.PNG)

> Subnets para os dados

5. Subnet app-az1
![Subnet app-az1](Images/VPC/Subnets/app-az1.PNG)

6. Subnet app-az2
![Subnet app-az2](Images/VPC/Subnets/app-az2.PNG)

> Subnets para as intâncias

### Criação das Route Tables
1. Criar route table privada
No console da AWS vá em VPC > Route tables > Create route table
![PRIVATE ROUTE TABLE](Images/VPC/RouteTables/private-rt-settings.PNG)

2. Criar route table pública
No console da AWS vá em VPC > Route tables > Create route table
![PUBLIC ROUTE TABLE](Images/VPC/RouteTables/public-rt-settings.PNG)