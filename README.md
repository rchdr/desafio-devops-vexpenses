# PROCESSO SELETIVO | ESTÁGIO EM DEVOPS

## Análise Técnica do Código Terraform

``` bash
provider "aws" {
  region = "us-east-1"
}
```
- Primeiro, é declarada qual Cloud Provider é para ser usada, AWS. Como também o datacenter em que será armazenado a aplicação, us-east-1.

``` bash
variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}
```
- Duas variáveis do tipo string são declaradas (projeto e candidato), suas descrições e valores default.

``` bash
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}
```
- Recurso que cria uma chave privada segura, e que ficará armazenada no Terraform state file. A chave será do formato RSA, com o tamanho de 2048 bits.

``` bash
resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
``` 
- Prover um recurso de EC2 key pair, que é usado para controlar o login em instâncias EC2.

``` bash
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
```
- Recurso que inicia uma VPC (Virtual Private Cloud) da Amazon. Uma rede virtual isolada logicamente.

``` bash
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}
``` 
- Recurso que cria uma subnet para a VPC. Subnet, que é um intervalo de endereços IP.

``` bash
resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}
``` 
- Recurso que cria uma VPC Internet Gateway. Componente que permite a comunicação entre a VPC e a internet.

``` bash 
resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}
```
- Recurso que cria uma route table para a VPC. Conjunto de regras, chamado de rotas, que determinam para onde o tráfego de rede de sua sub-rede ou gateway é direcionado.

``` bash
resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}
```
- Recurso que cria a associação entre a route table e a subnet.

``` bash
resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de qualquer lugar e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  # Regras de entrada
  ingress {
    description      = "Allow SSH from anywhere"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  # Regras de saída
  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}
``` 
- Recurso que cria o grupo de segurança, e explica como a entrada e saída de dados deve ser feita.

``` bash
resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
```
- Recurso que cria a instância EC2, e utiliza o debian como sistema operacional.

``` bash
data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}
```
- Data source usado para pegar o ID de uma AMI (Amazon Machine Image) registrada do debian, que é guardado como debian12.

``` bash
output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}
output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```
- Por último, declarados os outputs. Valores sobre a infraestrutura que serão exibidos na linha de comando. Neste caso, o IP público e chave privada da instância EC2.

## Modificação e Melhoria do Código Terraform

De acordo com as boas práticas, devemos separar os block types e as variáveis em arquivos diferentes. Criei, então: settings.tf, data.tf; outputs.tf; providers.tf; variables.tf; e as pastas environment/dev, onde ficará o arquivo terraform-dev.tfvars no qual contém os valores das variáveis.
Como o terraform-dev.tfvars não está no mesmo diretório do main, será necessário referenciá-lo na hora do terraform plan e terraform apply.
``` bash
$ terraform plan -var-file=environment/dev/terraform-dev.tfvars
$ terraform apply -var-file=environment/dev/terraform-dev.tfvars
```

Para uma melhor segurança, não armazenar o terraform.tfstate localmente. Nesse caso iremos utilizar o AWS S3. O seguinte código ficará no arquivo settings.tf
``` bash
terraform {
  backend "s3" {
    bucket         = "s3-bucket"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "TerraformStateLocking"
  }
}
```

Adicionando as duas linhas seguintes no parâmetro user_data do recurso aws_instance, o NGINX será instalado automaticamente quando a instância EC2 for criada, como também um arquivo index.html com uma tag h1, que ficará no diretório web padrão do NGINX. 
``` bash
apt-get install -y nginx
echo "<h1>Desafio Devops</h1>" > /var/www/html/index.html
```


## Instruções de Uso

- Para utilizar este código é necessário ter uma conta na AWS e ter o plugin Terraform da AWS instalado previamente. Se não, basta copiar o código abaixo no início do arquivo main.tf.
``` bash
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "5.72.1"
    }
  }
}
```

- Para iniciar, planejar e aplicar o código.
``` bash
$ terraform init
$ terraform plan -var-file=environment/dev/terraform-dev.tfvars
$ terraform apply -var-file=environment/dev/terraform-dev.tfvars
```

- Para excluir a aplicação criada.
``` bash
$ terraform destroy
```

## Agradecimento
Gostaria de agredecer a VEXPENSES por esta oportunidade, mesmo que eu não passe no processo seletivo, a experiência foi bem bacana e produtiva.