# DERS 5
# Create “webserver”  module

* Root main.tf deki module "aws_security_group" "default-sg" ve alt tarafınındaki bütün kodu kes ve modules/webserver/main.tf in içine yapıştır . Webserver/main.tf i de aşağıdaki koddaki gibi güncelle:

```bash
resource "aws_default_security_group" "default-sg" {
    vpc_id = var.vpc_id # DEĞİŞTİ

    ingress{
        from_port = 22
        to_port = 22
        protocol = "tcp"
        cidr_blocks = [var.my_ip]
    }
    ingress {
        from_port = 8080
        to_port = 8080
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]

    }

    egress {
        from_port = 0
        to_port = 0
        protocol = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
    tags = {
        Name: "${var.env_prefix}-default-sg"
    }
}

data "aws_ami" "latest-amazon-linux-image" {
    most_recent = true
    owners = ["amazon"]
    filter {
        name = "name"
        values = [var.image_name] # DEĞİŞTİ
    }    
    filter {
        name = "virtualization-type"
        values = ["hvm"]
    } 
}

resource "aws_key_pair" "ssh-key" {
    key_name = "key-pair"
    public_key = var.my_public_key
}
resource "aws_instance" "myapp-server"{
    ami = data.aws_ami.latest-amazon-linux-image.id 
    instance_type = var.instance_type

    
    subnet_id = var.subnet_id # DEĞİŞTİ
    vpc_security_group_ids = [[aws_default_security_group.default-sg.id]# DEĞİŞTİ
    availability_zone = var.avail_zone
 
   
    associate_public_ip_address = true
    key_name = aws_key_pair.ssh-key.key_name

    user_data = file("entry-script.sh")

    tags = {
      Name: "${var.env_prefix}-server"
    } 
}
```

* entry-script.sh I modules/webserver ın içine taşı.

* webserver/variables.tf i aç ve şu variable ları içine oluştur.

```bash
variable vpc_id{}
variable my_ip{}
variable env_prefix{}
variable image_name{}
variable public_key_location{}
variable instance_type{}
variable subnet_id{}
variable avail_zone{}
variable env_prefix{}
```

* Güzel, variable larımızı oluşturduk. Şimdi de variable larımızı root main.tf'te refer edelim:

```bash
module "myapp-webserver" {
    source = "./modules/webserver"
    vpc_id = aws_vpc.gelistirici-vpc.id
    my_ip = var.my_ip
    env_prefix = var.env_prefix
    image_name = var.image_name
    public_key_location = var.public_key_location
    instance_type = var.instance_type
    subnet_id = module.myapp-subnet.subnet.id
    avail_zone = var.avail_zone
}
```

* terraform.tfvars a şunu ekle: image_name = "amzn-ami-hvm-*-x86_64-gp2".

```bash
image_name = "amzn-ami-hvm-*-x86_64-gp2"
```

* Root output.tf I şu şekilde güncelle:

```bash
output "ec2_public_ip" {
    value = module.myapp-server.instance.public_ip
}
```

* modules/webserver output.tf i şu şekilde güncelle:

 ```bash
output "instance" {
    value = value = aws_instance.myapp-server
}
```

* İhtiyacımız olan herşeye sahibiz. Şimdi kodu çalıştırabilirsin. Cmd ye
        •	terraform init
        •	terraform plan
        •	terraform apply –auto-approve

* Böylece modülleri de tamamlamış olduk arkadaşlar

# DEMO PROJECT -2 
# Terraform & AWS EKS

* Bu kısımda Terraform kullanarak EKS Cluster oluşturacağız. Bunun için

1)	Control Planı oluşturma. 
    a.	mycluster.eks.aws.amazon.com
    b.	Master Nodes

2)	VPC Oluşturma
    a.	Worker Node ların çalışması için VPC leri oluşturuyoruz.
    b.	EC2 Instances
    c.	Node Group
-	Cluster ı spesific bir region da oluşturmamız gerekiyor(Bizim durumumuzda us-east-1)

#  VPC

* vpc.tf isimli bir dosya oluştur.
* terraform.tfvars isimli bir dosya oluştur.
  
* Root main.tf i ve modules I silelim. Yeni oluşturacağımız dosyaları cloud formation ile oluşturacağız. Cloud formationu ECS ile daha uyumlu olduğu için kullanıyoruz. Bunun için de Cloudformation templete gibi birşeye ihtiyacımız var(Hazır olarak var bu). Cloudformation Terraforma da alternatif 
çünkü o da bir infrastructure provisioning tool ancak sadece AWS için. 

* registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest I aç. Provision Instructions un altındaki kodu kopyala ve oluşturacağın vpc.tf e yapıştır. Modülün ismini “mapp-vpc” olarak değiştir. Terraform init deyince otomatik olarak indirecek. Attribute ları tanımlamak suretiyle kodu aşağıdaki gibi doldur:

*  vpc_tf için:
```bash
# Değişkenler
variable vpc_cidr_block{}
variable private_subnet_cidr_blocks{}
variable public_subnet_cidr_blocks{}

module "myapp-vpc" {
    source = "terraform-aws-modules/vpc/aws"
    version = 3.11.0
    # insert the 21 required variables here

    name = "myapp-vpc" 
    cidr = var.vpc_cidr_block
    # Best Practice: Her bir availability zone iççin bir private bir de public subnet oluştur;
    private_subnets = var.private_subnet_cidr_blocks
    public_subnets = var.public_subnet_cidr_blocks
}
```

* terraform.tfvars için
```bash
vpc_cidr_block = "10.0.0.0/16"
```
* Region'umuzda kaç Availability Zone(AZ) varsa hepsine bir private bir de public subnet cidr block atamamız gerekiyor. Bizim durumumuzda 6 adet AZ var. Bu noktada 6 private 6 public olmak üzere toplamda 12 adet subnet cidr block oluşturmamız AZ ye bu subnet leri dağıtmamız gerekiyor.Hatta bunu dinamic olarakta tanımlayabiliriz. Bunun için:

```bash
# (private ve public) subnet_cidr_blocks vpc_cidr_blocks 'un parçası olmalı
private_subnet_cidr_blocks = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24", "10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]
public_subnet_cidr_blocks = ["10.0.7.0/24", "10.0.8.0/24", "10.0.9.0/24", "10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]
``` 

* Birkaç tane daha attribute oluşturabiliriz:
    Cmd ye
    •	terraform init

* vpc.tf i şu şekilde doldur: 
```bash
provider "aws" { # YENİ
    # data 'nın bağımlı oldğu provider
    region = "us-east-2" # YENİ
} # YENİ

variable vpc_cidr_block{}
variable private_subnet_cidr_blocks{}
variable public_subnet_cidr_blocks{}

data "aws_availability_zones" "azs" {} # bu data AZ leri query leyecek. İsmi azs.
# "aws_availability_zones" "aws" provider ına bağlı olduğu için "aws" providerını da belirtmeliyiz.


module "myapp-vpc" {
    source = "terraform-aws-modules/vpc/aws"
    version = "3.11.0"
    

    name = "myapp-vpc" # Resource daki Name Sütunu
    cidr = var.vpc_cidr_block # Burada hard codingtense variable ları kullanacağız. cidr block umuzu tanımlamıştık zaten önceden.
    
    
    # Best Practice: Her bir availability zone iççin bir private bir de public subnet oluştur;
    # myapp-vpc modülünün içinde subnetler zaten çoktan tanımlı. Biz kaç tane subnet oluşturulacağını, hangi subnetlerin oluşturulacağını ve bu subnetlerde hangi cidr block larının oluşturulacağına karar vereceğiz.
    # EKS için de best practice her bir Availability Zone için bir adet private bir adette public subnet oluşturmak.
    # Bizim regionumuzda buradan da(https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Home:) görüleceği üzere 6 adet Availability Zone var. Bu durumda biz 6 adet A.Z. mizin her biri için bir adet public bir adet private subnet oluşturacağız.  
    private_subnets = var.private_subnet_cidr_blocks 
    public_subnets = var.public_subnet_cidr_blocks
    # Variable'ları yukarıya tanımladık.
    # Tanımladığımız variable ların değerlerini de terraform.tfvars a atadık.
    azs = data.aws_availability_zones.azs.names # dinamik tanımlama için variablelar ile module "myapp-vpc" nin arasına "azs" isimli datamızı oluşturduk.
    # data.aws_availability_zones un names attribute u olduğunu nereden biliyoruz peki ? Buradan arkadaşlar: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones

    enable_nat_gateway = true # defaule değeri her bir subnet için bit NAT gateway i. Transparency purposes için true ya set ettik.
    single_nat_gateway = true # bütün private subnetler single_nat_gateway aracılığılıyla internet trafiklerini yönlendirirler.
    enable_dns_hostnames = true # bununla da oluşturacağımız EC2 instance ını public & private dns hostnames e de atayacağız.

    # Tag lerimizi de ekleyelim:
    # Tag lerimiz kolay okunma faydasınının yanı sıra Cloud Controller Manager Identifier'a hangi resource ile bağlantıda olması gerektiğini hatırlatır. 
    
    # Aşağıdaki tag leri tanımlamak zorunlu
    tags = { # VPC için
    # myapp-eks-cluster cluster adı olsun.
    "kubernetes.io/cluster/myapp-eks-cluster" = "shared" # value = shared
    }

    public_subnet_tags = { # public_subnets için
    # myapp-eks-cluster cluster adı olsun.
    "kubernetes.io/cluster/myapp-eks-cluster" = "shared" # value = shared
    "kubernetes.io/role/elb" = 1 # elb = elastic load balancer
    # Kubernetis te load balancer service oluşturduğumuzda service'e cloud native load balancer atar.  Bunu da public subnet yoluyla yapar çünkü load balancer cluster için bir giriş noktasıdır. Bu yüzden dışarıya erişime açık olmalıdır. Load balancer external IP adress alır ve dışarıyla iletişimi bu IP adresi üzerinden yapar. İkinci dependency ile de bunu yaparız aslında; Kubernetis e iletişime geçmesi gereken dışarıya açık public subnet in bu olduğunu söyleriz. 
    }

    private_subnet_tags = { # private_subnets için
    # myapp-eks-cluster cluster adı olsun.
    "kubernetes.io/cluster/myapp-eks-cluster" = "shared" # value = shared
    "kubernetes.io/role/internal-elb" = 1 # Private subnet internete kapalıdır.
    }
}
```  
# Terraform & AWS EKS

* eks-cluster.tf isimli yeni bir dosya oluştur. Bu sefer eks module u kullanacağız. https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest a git. Dependencies'i aç Bu adımda kubernates provider I yapılandırmalıyız. Gördüğünüz gibi bu module kubernetise bağımlı.Bunu da şu şekilde yapacağız:

* eks-cluster.tf

```bash
provider "kubernetes" { # 8)
    
    host = data.aws_eks_cluster.myapp-cluster.endpoint # (Aşağısı) Endpoint of K8 cluster(API Server) host'umuza query lerek ulaşalım böylece harcode etmemiş oluruz & daha dinamik olur. Bu yüzden de aşağıya data "aws_eks_cluster" "myapp-cluster"ı tanımlyalım. 

    # Actual Authentication Data
    token = data.aws_eks_cluster_auth.myapp-cluster.token # token ları içeren objectleri istiyoruz burada 
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.myapp-cluster.certificate_authority.0.data) # certificate_authortity.0.dat bize cluster_ca_certificate i verecek. data.aws_eks_cluster.myapp-cluster da zaten query nin döndüreceği object. 
    # base64decode u kullandık çünkü certificate_authority encoded(https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/eks_cluster); biz bunu decoded istiyoruz.
}

data "aws_eks_cluster" "myapp-cluster" { # 9)
    name = module.eks.cluster_id # host'a queryleyrek ulaşmak için. Modülümüzü refer ettik. 
}

data "aws_eks_cluster_auth" "myapp-cluster" { # 10) token'ı içeren object'i verir bize bu 
    name = module.eks.cluster_id # actual authentication data için kullanırız. Aynı methodla query le ama bize bu sefer aws_eks_cluster_auh u ver. 
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "17.24.0"
  # insert the 7 required variables here

  cluster_name = "myapp-eks-cluster" # 1) cluster 'ın ismi. Bunu zaten vpc.tf deki tags'te tanımlamıştık.
  cluster_version = "1.21" #2) kubernetis version

  # Subnet Attributes. Worker Noda ların başlamasını istediğimiz subnetler
  subnets = module.myapp-vpc.private_subnets # 3) Burada myapp-vpc modulündeki private_subnets'e refer ediyoruz. Hatırlayalım ki private subnets bizim workloadımız, public subnet ise dışarıyla iletişimi kuran external resources(loadbalancer gibi)
  vpc_id = module.myapp-vpc.vpc_id # 7) Burada da myapp-vpc deki vpc_id ye refer ediyoruz.  

  # 4) tags. eks cluster için gerekli taglarimiz ok, yani tanımlamazsak hata almayız. 
  tags = {
      environment = "deployment" # 5)
      application = "myapp" # 6) hangi application için
  }

  # Worker Nodes. Burada da hangi tür worker node larla çalışmak istediğimizi belirtelim.
  # 8
  worker_groups = [ #Takes array of worker node config objects as an input. Yani içerisine birden fazla worker node tanımlayabiliriz. 
      {
          instance_type = "t2.small" # t2.small türünde
          name = "worker-group-1" # isimli
          asg_desired_capacity = 2 # 2 adet worker nodes istiyoruz.
      }, # ne zaman bir eks cluster oluştursak, AWS bize oluşturduğumuz her bir cluster için ayrı bir fatura çıkartır. - Saatlik 10 cent - + 10 cent for each worker nodes. Bu yüzden işin bittikten sonra işe yaramayan component ları destroy etmeyi unutma!!
      {
          instance_type = "t2.medium" # t2.medium türünde
          name = "worker-group-2" # isimli
          asg_desired_capacity = 1 # 1 adet worker node istiyoruz.
      }
  ]
}
``` 

* Cmd ye git ve:
    * terraform init 
    •	terraform plan
    •	terraform apply --auto-approve(bu süreç 10-15 dk arası sürebilir)
    * Hata alırsan terraform destroy deyip tekrar deneyebilirsin.
  

* AWS ten region nunu us-east-2 yap(vpc'ni oraya tanımladın çünkü) 
* Arama tuşuna da AWS/EKS/Clusters( 
https://us-east-2.console.aws.amazon.com/eks/home?region=us-east-2#/clusters) yaz. Gördüğünüz gibi myapp-eks-cluster orada Tıkla üzerine. Bunlar bizim oluşturduğumuz node lar arkadaşlar--eks-cluster.tf'deki worker nodes a gel-- gördüğünüz gibi 2 adet small 1 adet medium worker node oluştur demişiz. AWS e geri gel. ve oluşmuş arkadaşlar. Keşfet geri kalanını da.
* IAM Dashboard'a gel. Roles a gir. Arama tuşuna myapp yaz. Gördüğünüz gibi 2 adet rol oluşturulmuş arkadaşlar. Biri eks diğeri de ec2 için. Bunu kodladığımı EKS Modülüyle otomatik olarak oluşturduk arkadaşlar. 
* EC2 Instances a gel. Bunlar da oluşturduğumuz instance lar arkadaşlar. 
* Your VPC s e gel. myapp-vpc de bizim oluşturduğumuz VPC arkadaşlar- İçine gir ve keşfet. Route Table larından myapp-vpc olanları filtrele. myapp-vpc-private'e tıkla. Görmüş olduğun nat Worker Node'ların Master Node'lara bağlanmasını sağlıyor arkadaşlar. Bu nat'a ihtiyacımız var çünkü worker node larımız ve master node larımız ayrı vpc lerde ve birbirleriyle konuşmaları gerekiyor. AWS de bu iletişimi nat gateway ile sağlıyor. 
* security groups u da kontrol edelim. arama tuşuna myapp yaz ki filtrelensin. bunlar da worker node larımız için olan security groups arkadaşlar.  worker nodes larla master nodes arasındaki iletişimi secure eden de webserver-sg arkadaşlar

* eks.cluster.tf i aşağıdaki gibi doldur:

```bash
provider "kubernetes" {
    load_config_file = "false" # default config file'ı istemiyoruz.
    # (Aşağısı) Endpoint of K8 cluster(API Server)
    host = data.aws_eks_cluster.myapp-cluster.Endpoint

    # Actual Authentication Data
    token = data.aws_es_cluster_auth.myapp-cluster.token
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.myapp-cluster.certificate_authortity.0.data)
}

data "aws_eks_cluster" "myapp-cluster" {
    name = module.eks.cluster_id # host için
}

data "aws_eks_cluster_auth" "myapp-cluster" {
    name = module.eks_cluster_id # actual authentication data için
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "17.24.0"
  # insert the 7 required variables here

  cluster_name = "myapp-eks-cluster" # cluster 'ın ismi
  cluster_version = "1.17" # kubernetis version

  # Subnet Attributes. Worker Noda ların başlamasını istediğimiz subnetler
  subnets = module.myapp-vpc.private_subnets
  vpc_id = module.myapp-vpc.vpc_id

  # tags
  tags = {
      environment = "deployment"
      application = "myapp" # hangi application için
  }

  # Worker Nodes
  worker_groups = [
      {
          instance_type = "t2.small" # t2.small türünde
          name = "worker-group-1" # isimli
          asg_desired_capacity = 2 # 2 adet worker nodes istiyoruz.
      },
      {
          instance_type = "t2.medium" # t2.medium türünde
          name = "worker-group-2" # isimli
          asg_desired_capacity = 1 # 1 adet worker node istiyoruz.
      }
  ]
}
