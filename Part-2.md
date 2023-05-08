# DERS 4
# Ssh key pairi Variable Haline Getirme
* Aradaşlar server-key-pair gibi önemli bir dosyayı güvenlik ve dinamiklik açısından variable olarak kullanmak daha mantıklı.Bu yüzden:

* Öncelikle my_private_key variable ını main.tf e oluştur:

```bash
variable my_private_key{}
```


*  Bu değişken key pair'deki my_private_key'in konumunu verir.
  
* my_private_key e set ettiğimiz değeri arkadaşlar hatırlarsanız dün AWS key-pair'den oluşturmuştuk. Ben "server-key-pair-2" olarak oluşturdum çünkü "server-key-pair" isimli bir key pairim var. Sizin bu durumda benim "server-key-pair-2" yazdığım yerlere "server-key-pair-2" yazmanız gerekiyor.

* terraform-dev.tfvars a my_private_key değişkenini tanımla:
/*
```bash
my_private_key= "server-key-pair-2”
```

* Kodu şu şekilde güncelle:

```bash
resource "aws_instance" "myapp-server"{
    ami = data.aws_ami.latest-amazon-linux-image.id 
    instance_type = var.instance_type

    # Bu kısım opsiyonel, kodu optimize etmek için yapıyoruz.
    subnet_id = aws_subnet.myapp-subnet-1.id
    vpc_security_group_ids = [aws_default_security_group.default-sg.id]
    availability_zone = var.avail_zone
 
    # Bu kısım gerekli
    associate_public_ip_address = true

    # Aşağıda key-pairi aws_instance resource umuza bağladık.
    key_name = var.my_private_key # DEĞİŞTİ

    tags = {
      Name: "${var.env_prefix}-server"
    } 
}
```

* terraform plan yap.
  
* Sonrasında terraform apply --auto-approve yap.

* resuorce "myapp-server" nin altına yaz aşağıdakini:

```bash
output "ec2_public_ip" {
  value = aws_instance.myapp-server.public_ip
}
```

* Cmd ye terraform plan ve terraform apply –auto-approve yaz. AWS Instances I aç. Gördüğünüz gibi eskisini terminate etti yenisi çalışır vaziyette.


* Kodun son hali:

```bash

provider "aws" {
    region = "us-east-1"
}

variable vpc_cidr_block{}
variable subnet_cidr_block{}
variable avail_zone {}
variable env_prefix {}
variable my_ip{}
variable instance_type{}
variable my_private_key{} # DEĞİŞTİ


resource "aws_vpc" "myapp-vpc" {
    cidr_block = var.vpc_cidr_block
    tags = {
        Name: "${var.env_prefix}-vpc"
    }
}

resource "aws_subnet" "myapp-subnet-1" {
    vpc_id = aws_vpc.myapp-vpc.id
    cidr_block = var.subnet_cidr_block
    availability_zone = var.avail_zone
    tags = {
        Name: "${var.env_prefix}-subnet-1" # subnet-1 post-suffix arkadaşlar
    }
}


# Internet gateway Oluşturma
resource "aws_internet_gateway" "myapp-igw" {
    vpc_id = aws_vpc.myapp-vpc.id
    tags = {
        Name: "${var.env_prefix}-igw"
    }
}

resource "aws_default_route_table" "main-rtb" {
    default_route_table_id = aws_vpc.myapp-vpc.default_route_table_id

    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.myapp-igw.id
    }
    tags = {
        Name: "${var.env_prefix}-main-rtb"
    }
}

resource "aws_security_group" "default-sg" {
    vpc_id = aws_vpc.myapp-vpc.id

    # Firewall Kuralları
    # a) Incoming Traffic Rules: ssh into EC2 & access from browser
    # 1. SSH
    ingress{
        from_port = 22
        to_port = 22
        protocol = "tcp"
        cidr_blocks = [var.my_ip]
    }

    # 2. Firewall
    ingress { # ingress for incoming
        from_port = 8080
        to_port = 8080
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"] 

    }

    # b) Outgoing Traffic Rules: installations & fetch Docker Image
    egress { 
        from_port = 0
        to_port = 0
        protocol = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
    tags = {
        Name:"${var.env_prefix}-default-sg"
    }
}

# AMI değerine hardcode etmeden query leyerek ulaşma
data "aws_ami" "latest-amazon-linux-image" {
    most_recent = true
    owners = ["amazon"]
    filter {
        name = "name"
        values = ["amzn-ami-hvm-*-x86_64-gp2"] 
    }    
    filter {
        name = "virtualization-type"
        values = ["hvm"]
    } 
}

#Filterlarımızın output u neymiş bakalım
output "aws_ami_id" {
    value = data.aws_ami.latest-amazon-linux-image.id
}


resource "aws_instance" "myapp-server"{ 
# Bu noktada 2 adet instance oluşturmamız gerekiyor:
    ami = data.aws_ami.latest-amazon-linux-image.id # EC2 server'ının dayandığı image
    instance_type = var.instance_type

    
    # Bu kısım opsiyonel, kodu optimize etmek için yapıyoruz.
    subnet_id = aws_subnet.myapp-subnet-1.id
    availability_zone = var.avail_zone
    
    # Bu kısım gerekli
    associate_public_ip_address = true # ip adreslerine browserdan eriş
    key_name = var.my_private_key  # DEĞİŞTİ

    tags = {
      Name: "${var.env_prefix}-server"
    } 
}


output "ec2_public_ip" { # DEĞİŞTİ
  value = aws_instance.myapp-server.public_ip # DEĞİŞTİ
} # DEĞİŞTİ
```

* Cmd ye terraform state show aws_instance.myapp-server yaz. 
  
*  Böylece key-pair değerleri için de otomasyonumuzu yapmış olduk. 



* EC2 server ımız çalışır vaziyette, Networking kısmını ayarladık Ancak server’ımızın içinde henüz hiçbir şey çalışmıyor. Docker I yüklemedik, docker containerlar çalışmıyor. Server ımız boş şu an. Yapmadığımız şeyleri de automated yolla yapmamız gerekiyor. Instance larımız hazır olur olmaz bir dizi komutla her şeyi halletmek istiyoruz. Bunu da(resource “aws_instance” “myapp-server”ın içine):


```bash
resource "aws_instance" "myapp-server"{    
    .
    .
    .
    key_name = var.my_private_key

    user_data = <<EOF
                    #!/bin/bash
                    sudo yum update -y && sudo yum install -y docker
                    sudo systemctl start docker
                    sudo usermod -aG docker ec2-user
                    docker run -p 8080:80 nginx
                EOF
    /*
    Üstteki Kısım Linux Komutu
    ilk satır: bash script
    ikinci satır: bütün paketleri ve repoları günceller && güncellemeden sonra docker ı yükle
    Üçüncü Satır: yüklenen docker ı çalıştır
    Dördüncü Satır: Docker komutlarını sudo komutu olmadan çalıştırmak istiyoruz--
    -- Bunun için de EC2 kullanıcısını docker group a eklemeliyiz
    Beşinci Satır:  Docker ı çalıştırır

    ** Bu kısım yalnızca bir kere çalışır.
    */
    ```
    tags = {
      Name: "${var.env_prefix}-server"
    }
}
```

* Cmd ye terraform plan yaz.  Resource un birini yok etmiş birini oluşturmuş

* Cmd ye terraform apply –auto-approve yaz. ami_id mizi AWS ten bakmamıza gerek kalmadan console a yazdırdı.--bunu zaten önceden belirlediğimiz output ile yapmıştık-- Bu arada EOF un içindekiler sadece bir kere çalıştırılıyor aklında bulunsun. 

* Görüldüğü üzere consoldaki ami değerimiz üzerinde çalıştığımız Instance(dev-server) uyuşuyor. 
  
## Demo Project 1 i ana hatlarıyla tamamladık arkadaşlar, şimdi de optimize edelim.

# Extract to shell script

* Elimizde daha karmaşık bir scriptin olduğu durumlarda EOF kısmını main.tf dense ayrı bir dosyada saklamalıyız. entry-script.sh i oluştur.

* user_data yı şu şekilde güncelle:

```bash
user_data = file("entry-script.sh")
```

* Cmd ye terraform plan ve terraform apply –auto-approve yaz. 

# Configuring Infrastructures, not Servers
Infrastructure oluşturulduğu, yönetildiği ve configure edildiği zaman terraformun işi biter. Docker yüklemek falan terraformun görev alanına girmiyor. 

![terraform-manages-infrastructure](terraform-manages-infrastructure.png)

entry-script.sh in içindeki kısım shell scripting. Shell Script de bilmekte fayda var.

 ![cmt](cmt.png)

* Teffaform sadece ilk üç adımı kapsar. Bu yüzden projemizin kalanında Ansible la gideceğiz ki çoğunlukla Ansible tercih edilir. 


# PROVISIONERS

* Entry-script.sh terraform un görev alanına girmiyor. Bu yüzden burada alacağımız hatayı açıklayamaz 
ve bu da işlerin bizim için daha zor olmasını sağlar. Terraform dan komutları çalıştırmanın başka yolları da var. Bu yollar da terraform un kapsama alanına girmiyor ama bilmekte fayda var. 
* Provisioner bunlardan birisi
* "myapp-server" ın içine tags in tam üstüne bir provisioner tanımla
  
```bash
resource "aws_instance" "myapp-server"{
    .
    .
    .
    provisioner "remote-exec" {
        # remote server a bağlanmamızı sağlayan provisioner
        inline = [ # remote server'da çalıştırılacak komutları içine tanımlarız.
            "export ENV_dev",
            "mkdir newdir"
        ]
    }
```

* ne zaman provisioner "remote-exec"i oluşturacak olursak terraforma ayrıca remote server a nasıl bağlanması gerektiğini de söylemeliyiz.

```bash
    connection { # YENİ
    # ne zaman provisioner "remote-exec"i oluşturacak olsa terraforma aynı remote server a nasıl bağlanması gerektiğini de söylemeliyiz.
        type = "ssh" # YENİ
        host = self.public_ip # YENİ
        user = "ec2-user" # YENİ
        private_key = var.my_private_key # YENİ
    } # YENİ

    provisioner "remote-exec" { # YENİ ...
```

* Cmd ye terraform plan ve terraform apply –auto-approve yaz. Uygulamada script le devam etmek daha mantıklı bu yüzden : 

```bash
provisioner "remote exec" {
    # remote server a bağlanmamızı sağlayan provisioner
#   inline  [ # remote server'da çalıştıracak komutları içine tanımlarız.
#       "export ENV=dev",
#       "mkdir newdir"
#   ] 

# Uygulamada inline bloğundansa scriptt ile devam etmek daha mantıklı:
script = file("entry-script.sh")
}
```

Bir provisioner daha oluşturacağız: 

```bash
provisioner "file" {
    # dosyaları veya directory leri local dan eni oluşturulan resource a kopyalar.
    source = "entry-script.sh"
    destination = "entry-script.sh" # cmd ye pwd yaptığımız zaman çıkan yer
}

provisioner "local-exec" {
    # bir kaynak oluşturulduktan sonra yerel bir yürütülebilir dosyayı çağırır
    command = "echo ${self.public_ip} >output.txt" # yukarıdaki server'ın public IP adresini cmd'ye yazdırmak ve output.txt'ye kaydetmek için
}
```

*  Provisionları kullanmak terraform tarafından önerilmiyor çünkü:
        -	use user_data if avaliable
        -	breaks idempotency concept
        -	TF doesn’t know what you are execute
        -	Breaks current-desired state comparison
        -	Ayrıca dengesizdirler; bazen çalışırlar bazen çalışmazlar. Kodun bazı kısımlarını okumayabilirler.
* Onun yerine remote exec durumları için:
        -	user_data yı kullan.
        -	Server provision edildikten sonra Configuration Management Toolların kullan. Chef, Puppet, Ansible gibi
        -	“local-exec” provisioner in yerine mesela HasHiCorp un “local” provider kullanabilirsin. (https://registry.terraform.io/providers/hashicorp/local/latest/docs/resources/file)
        -	Execute scipts separate from Terraform
        -	From CI/CD tool


* Ama her halükarda başkasının config dosyasında mesela provisioner kullanılmış, ne yapmanız gerektiğini bilmeniz gerekiyor. O yüzden öğrenmekte fayda var.

* Cmd ye terraform plan ve terraform apply – auto-approve yaz. Bir şey değişmedi arkadaşlar çünkü entry-script.sh imiz aynı dosya. Provisioner burada bize ne sağladı? entry-script.sh i daha yönetilebilir kıldı. Eğer entry-script.sh de bir hata olsaydı provisioner sayesinde hatayı cmd de görebilecek önlemimizi ona göre alacaktık. 

* Kodun son hali: 
```bash

provider "aws" {
    region = "us-east-1"
}

variable vpc_cidr_block{}
variable subnet_cidr_block{}
variable avail_zone {}
variable env_prefix {}
variable my_ip{}
variable instance_type{}
variable my_private_key{}


resource "aws_vpc" "myapp-vpc" {
    cidr_block = var.vpc_cidr_block
    tags = {
        
 # Her bir component için deploy edildikleri enviromentları ön ad olarak verelim. Bunun için de yukarıya env_prefix variable ını oluşturduk.

        Name: "${var.env_prefix}-vpc" # string interpolatation
    }
}

resource "aws_subnet" "myapp-subnet-1" {
    vpc_id = aws_vpc.myapp-vpc.id
    cidr_block = var.subnet_cidr_block
    availability_zone = var.avail_zone
    tags = {
        Name: "${var.env_prefix}-subnet-1" # subnet-1 post-suffix arkadaşlar
    }
}


# Internet gateway Oluşturma
resource "aws_internet_gateway" "myapp-igw" {
    # custom için internet gateway oluşturuyoruz şuan
    # Bunu sanal bir modem olarak düşünebiliriz; bizi internete bağlıyor
    vpc_id = aws_vpc.myapp-vpc.id
    tags = {
        Name: "${var.env_prefix}-igw"
    }
}

resource "aws_default_route_table" "main-rtb" {
    # cmd ye terraform state show aws_vpc.myapp-vpc yazmamızdan sonraki kısım
    default_route_table_id = aws_vpc.myapp-vpc.default_route_table_id

    # route ve tags kısmını myapp-route table dan yapıştırdık.
    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.myapp-igw.id
    }
    tags = {
        Name: "${var.env_prefix}-main-rtb"
    }
}

resource "aws_security_group" "default-sg" {
    vpc_id = aws_vpc.myapp-vpc.id
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
        Name:"${var.env_prefix}-default-sg"
    }
}

# AMI değerine hardcode etmeden query leyerek ulaşma
data "aws_ami" "latest-amazon-linux-image" {
    most_recent = true 
    owners = ["amazon"]
    filter {
        name = "name"
        values = ["amzn-ami-hvm-*-x86_64-gp2"] 
    }    
    filter {
        name = "virtualization-type"
        values = ["hvm"]
    } 
}

output "aws_ami_id" {
    value = data.aws_ami.latest-amazon-linux-image.id
}


resource "aws_instance" "myapp-server"{ 
    ami = data.aws_ami.latest-amazon-linux-image.id 
    instance_type = var.instance_type
    subnet_id = aws_subnet.myapp-subnet-1.id
    availability_zone = var.avail_zone

    # Bu kısım gerekli
    associate_public_ip_address = true 
    key_name = var.my_private_key
    user_data = file("entry-script.sh")

    connection { # YENİ
        type = "ssh" # YENİ
        host = self.public_ip # YENİ
        user = "ec2-user" # YENİ
        private_key = var.my_private_key # YENİ
    } # YENİ

    provisioner "remote-exec" { # YENİ
    /*
    # remote server a bağlanmamızı sağlayan provisioner
    inline = [ # remote server'da çalıştırılacak komutları içine tanımlarız.
        "export ENV_dev",
        "mkdir newdir"
    ]
    */
    script = file("entry-script.sh") # YENİ
    } # YENİ

    provisioner "file" { # YENİ
    source = "entry-script.sh" # YENİ
    destination = "entry-script.sh" # YENİ
}

    provisioner "local-exec" { # YENİ
        command = "echo ${self.public_ip} >output.txt" # YENİ
    } # YENİ

    tags = { # YENİ
      Name: "${var.env_prefix}-server" # YENİ
    }  # YENİ
} # YENİ


output "ec2_public_ip" { # YENİ
  value = aws_instance.myapp-server.public_ip # YENİ
} # YENİ

```


# MODULES

* Config dosyası oluşturduk. EC2 instance ı da tanımladık. Bunlar görece basit kodlar olmasına rağmen 100 satırdan fazla kod yazdık. Bunu da tek bir main.tf dosyası oluşturup bütün kodu onun içine yazdık. Bu kullanılabilir değil arkadaşlar, programlama mantığına da ters. Bundansa kodumuzu modüllere ayırmalı ve bu modülleri birbirine bağlamalıyız. Modülleri fonksiyonlara benzetebiliriz, ya da class lara. Bu metaforumuzda: 
            •	Input Variables mesela fonksiyon argümanlarına
            •	Output Values da fonksiyonların return değerine benzetilebilir. 

* Whenever we create a module, it should be the proper abstraction of resources. Yani birdense birden fazla resource için bir module oluşturmalı, resourceları bu modülün içine gruplamalıyız. Mesela VPC isimli bir modül oluşturacak olursak: subnet, internet gateway ve route table I kapsamalı. Bunlardan minimum 3-4 ünü yalnızca birini değil. 
  
* Kendi modülümüzü oluşturabiliriz, ama çoğu durum için Terraform un hali hazırda bir modülü var. Bu yüzden öncelikle bu modülleri bulmalı, incelemeli, amacımıza hizmet etmiyorlarsa kendi modülümüzü oluşturmalıyız. Terraform registy ye gidecek olursak(registry.terraform.io/browse/modules) modülleri daha yakından görebiliriz. Vpc modülüne gidelim mesela. Oldukça güzel bir arayüz, toplu bir data. Parameterse a erişebiliyoruz. Açıklamaları yazıyor. Çoğu durumda bunlardan birkaçını kullanırız. Kullanmadıklarımız default değerleriyle çalışır. Dependency kısmımız da var modülümüzde. Bu section da modülümüz başka providerlardan neleri refer ediyor onu görebiliriz.  Vpc modülünün aws dependency si var mesela. Bu demek oluyor ki ne zaman VPC modülünü kullansak sistem arkada AWS I de import edip çalıştırır. 

# Modularize Our Project

* Main.tf I modülize etmeye başlayabiliriz. 

* Yeni modülümü oluşturmadan önce main.tf I bi temizleyelim. Bu noktada 4 adet terraform dosyası oluşturacağız:
-	main.tf
-	variables.tf : (variable)değişkenleri buraya taşıyoruz
-	outputs.tf: output dosyalarını buraya taşıyoruz.
-	providers.tf: providerları buraya taşıyoruz.

* Yukarıdaki tf dosyalarını oluştur ve açıklamalardaki gibi taşı(providers hariç çünkü bir tane provider ımız var; sadece bunun için yeni bir dosya oluşturmanın anlamı yok). Terraform un iyi bir yanı bu dosyaları birbirine bağlamak için kod yazmamıza gerek olmaması. Otomatik olarak terraform bağlıyor zaten.

# Create a Module

* modules isimli bir klasörün oluşturalım. Bu klasörün içine de asıl modülleri içeren klasörler oluşturacağız. webserver ve subnet isimli klasörleri içine oluşturduk. Her bir modülün kendine ayrı main.tf outputs.tf ve variables.tf dosyaları olacak. Bu dosyaları VS Code Terminalinden oluşturacağız çünkü daha kolay. VS Terminal e:
    - cd modules
    - cd webserver
    - New-Item main.tf
    - New-Item variables.tf
    - New-Item outputs.tf  
    - cd ../subnet
    - New-Item main.tf
    - New-Item variables.tf
    - New-Item outputs.tf  

* Şöyle bi toparlayalım. Root module umuz ana main.tf file ımız. Children modulelar da modules klasörünün içindekiler. 
  
* (“aws_subnet”, “myapp-subnet-1”), (“aws_internet_gateway”, “myapp-igw”), (“aws_default_route_table”, “main-rtb”) resource larını kesip modules/subnet/main.tf e yapıştıralım. Içindeki bazı değerler root modülü için uyarlandığından dolayı onları alttaki fotodaki haline güncelleyelim.

```bash
modules/subnet/main.tf

resource "aws_subnet" "myapp-subnet-1" {
    vpc_id = var.vpc_id
    cidr_block = var.subnet_cidr_block
    availability_zone = var.avail_zone
    tags = {
        Name: "${var.env_prefix}-subnet-1"
    }
}

resource "aws_internet_gateway" "myapp-igw" {
    vpc_id = var.vpc_id
    tags = {
        Name: "${var.env_prefix}-igw"
    }
}

resource "aws_default_route_table" "main-rtb" {
    default_route_table_id = var.default_route_table_id
    
    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gatewa.myapp-igw.id
    }

    tags = {
        Name: "${var.env_prefix}-main-rtb"
    }
}
```

Ana variables.tf den subnet_cidr_block{}, avail_zone{} ve env_prefix{} variablelarını kopyala ve modules/subnet/variables.tf e yapıştır. Ayrıca vpc_id ve default_route_table_id variable larını da aynı dosyanın içine oluştur. 

```bash
modules/subnet/variables.tf
variable subnet_cidr_block {}
variable avail_zone {}
variable env_prefix {}
variable vpc_id {}
variable default_route_table_id {}

```

# Use the Module
* Main.tf imiz var, variablelar da variables.tf in içinde tanımlı. Peki subnet/main.tf imizi root main.tf den nasıl refer ederiz? module keyword’üyle tabiki. Alttaki kodu root main.tf in içine myapp-vpc & 
default-sg’nin arasına yazıyoruz.

```bash
module "myapp-subnet" {
    source = "./modules/subnet"
    subnet_cidr_block = var.subnet_cidr_block # modulün kaynağı için
    # main.tf den refer edeceğimiz variable'lar
    avail_zone = var.avail_zone
    env_prefix = var.env_prefix
    
    # tanımladığımız variable lar
    vpc_id = aws_vpc.myapp-vpc.id
    default_route_table_id = aws_vpc.myapp-vpc.default_route_table_id
}
```

# Module Output

* Child module daki resource lara nasıl erişebiliriz? subnet/outputs.tf i aç. 
* Output Values module un return değeri gibidir. Resource değerlerini parent module e taşır. 
```bash
output "subnet" {
    value = aws_subnet.myapp-subnet-1
}
```

* Root main.tf i aç. resource "aws_instance" "myapp-server" daki subnet_id yi şu şekilde güncelle: 

```bash
resource "aws_instance" "myapp-server"{
    ami = data.aws_ami.latest-amazon-linux-image.id 
    instance_type = var.instance_type

    
    subnet_id = module.myapp-subnet.subnet.id # DEĞİŞTİ
    vpc_security_group_ids = [aws_default_security_group.default-sg.id]
    availability_zone = var.avail_zone
 
   
    associate_public_ip_address = true
    key_name = aws_key_pair.ssh-key.key_name

    user_data = file("entry-script.sh")

    tags = {
      Name: "${var.env_prefix}-server"
    } 
}
```

# Apply Configuration Changes

* Öncelikle 
```bash
terraform init
``` 
yazalım çünkü ne zaman bir modül eklenirse/ değişirse terraform init kullanırız. Çalıştı. 

* Şimdi de
```bash 
terraform plan
``` 
yazalım.

```bash
* Terraform apply –auto-approve 
```
yaz.

* AWS Instances I aç. Gördüğümüz çıktılar(aws_ami_id ve ec2_public_ip) root module(main.tf) un outputları
