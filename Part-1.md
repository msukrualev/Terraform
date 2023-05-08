# Demo PROJECT-1

* AWS e EC2 instance I deploy edeceğiz. İçinde de ngix Docker container çalıştıracağız. Ama öncelikle AWS ten infrastructure inşaa edeceğiz. Buna da:git 
  - Custom bir VPC oluşturarak -1-
  - VPC nin içine custom bir subnet(birden fazla da inşaa edebiliriz) inşa ederek -2-
  - VPC leri de route table oluşturarak ve internet gateway kullanarak birbirlerine bağlayacağız -3-
  - VPC lerin içine de EC2 Instance ları inşaa edeceğiz.-5-
  - EC2 Instance ları da nginx Docker container larını çalıştıracak. -6-
  - İnşa ettiğimiz şeyleri browser üzerinden erişeceğimizden dolayı bir de Firewall oluşturacağız. Server üzerinden ayrıca SSH portu da açacağız. Bu portlar için de bir Security Group u oluşturacağız. -4-

![dp1](dp1.png)


* Best Practice
    -	Infrastructure u temelden oluştur
    -	AWS tarafından oluşturulmuş default değerleri olduğu gibi bırak

* Hadi başlayalım!

# Custom VPC yi oluşturma ve 2. VPC nin içine custom bir subnet(birden fazla da inşaa edebiliriz) inşa etme

* Şimdi bu noktada kodu biraz güncelledik:

```bash

provider "aws" {
region = "us-east-1"

}

variable vpc_cidr_block{}
variable subnet_cidr_block{}
variable avail_zone {}
variable env_prefix {}

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
```

* terraform.tfvars ı da şu şekilde güncelledik:

```bash
vpc_cidr_block = "10.0.0.0/16"
subnet_cidr_block = "10.0.10.0/24"
avail_zone = "us-east-1b"
env_prefix = "dev"
```

* İlk aşamamız hazır gibi. Cmd ye gelip terraform init ve plan yazalım & enter a basalım. 

* Öncelikle AWS/Subnets e girip kaç adet subnet oluşturulduğuna bir bak. 16 ise iki tanesini silmen gerekecek. Ayrıca VPC lerini de kontrol et. Max 5 adet olabilir. O yüzden 5 tane varsa 1 tanesini sil çünkü bu kodla bir tane daha oluşturacağız. 

* Sonrasında cmd ye şunu yaz: terraform apply --auto-approve

* İlk iki aşamayı tamamladık. Şimdi de AWS websitesine gidelim (AWS Console kısmına). Arama kısmına vpc yazalım. dev-vpc yi görücez orda.. Bu bizim oluşturduğumuz vpc arkadaşlar. Ayrıca Subnets I de kontrol edelim(Yandan Subnets e tıkladık). Bakın bu bizim oluşturduğumuz subnet: dev-subnet-1.  dev-vpc nin VPC ID sinin üzerine tıklayalım.
  

# 	VPC leri de route table oluşturarak ve internet gateway kullanarak birbirlerine bağlama

* Şimdi burada bazı şeyleri AWS bizim yerimize default olarak oluşturmuş. Bunlar neler?
    -	(Main)Network ACL: Subnetlerimiz için güvenlik duvarı yapılandırması. Network ACL in üzerine tıkla. Inbound Rules: Gördüğünüz gibi hepsi all. Yani Network ACL default değerinde hiçbir şekilde internet trafiğini sınırlandırmamış arkadaşlar. 
    - * şu demek: Denies all inbound IPv4 traffic not already handled by a preceding rule (not modifiable).
    Tabiki bunları istediğimiz gibi değiştirebiliriz.

    -	(Main)Route Table:VPC n için sanal router gibi bir şey. Yani VPC içindeki internet trafiğinin nereye doğru ilerleyeceğini belirliyor. (Main) Route Table a tıkla. Routes (aşağıdaki) sekmesini aç. Gördüğünüz gibi tek bir route yapılandırılmış. Destination daki aralığa bakarsak arkadaşlar bunun da tanımlamış olduğumuz vpc_cidr_block değeri olduğunu görebiliriz. Target=  local yani her şey tanımlamış olduğumuz VPC nin içinde halloluyor; hiçbir şey dışarı çıkmıyor yani aslında internete bağlı değiliz. Dışarı çıkmıyor. Filtreyi kaldır. Default route table değerlerinden birine tıkla. Görüldüğü gibi iki adet route umuz var burada. 0.0.0.0/0 olana bak(vpc-0e0528f357490a0b6). Görüldüğü gibi arkadaşlar bunun target I local değil. Yani internete bağlı. Biz de route table ımızı bu şekilde güncelleyeceğiz.

* Şimdi de route table ı terraform da oluşturalım. Best Practice: Default componentları—bu componentlardan biri de rout table dır-- kullanmaktansa yenilerini oluştur. Bu noktada yeni bir route table oluşturacağız.

```bash
# Route Table Oluşturma
resource "aws_route_table" "myapp-route-table" {
    vpc_id = aws_vpc.myapp-vpc.id # id mizi tanımladık

    route {
        # Bunun içine de route table daki entry leri tanımlayacağız
        # 10.0.0.0/16 için route oluşturmamıza gerek yok çünkü zaten default tanımlanmış
        cidr_block = "0.0.0.0/0" # birinci attribute umuz
        
        # route table ımız için internet gateway id
        # default değerimizde yok bu yüzden oluşturmamız gerekecek
        # alttaki resource ile oluşturalım
        gateway_id = aws_internet_gateway.myapp-igw.id
    }
    tags = {

        # Bütün component ları tag edelim ki bizim tarafımızdan oluşturulduklarını daha rahat anlayalım
        Name: "${var.env_prefix}-rtb"
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
```

* “aws_internet_gateway” i “aws_route_table” ın içindeki gateway_id için kullanacağımızdan dolayı öncelikle    “aws_internet_gateway” I oluşturmamız gerekiyor. Ancak bunu “aws_internet_gateway” I alta yazarakta yapabiliyoruz. Önceliği terraform otomatik olarak veriyor. 

* Cmd ye terraform plan yazalım. 
 
* Sonrasında terraform apply --auto-approve da yazıp çalışmamızı apply edelim.

* AWS/ Route Table lara gel. dev-rtb yazan bizim manuel olarak oluşturduğumuz route table. Görüldüğü gibi default olan route table ı custom olarak oluşturduk. 
   
# Subnet Association with Route Table

* AWS/ Route Table lara gel. dev-rtb yazan bizim manuel olarak oluşturduğumuz route table. dev-rtb/subnet associations a tıkla. Subnet associations umuz yok görüldüğü gibi.  Bunu değiştirelim. En alta bir resource daha tanımlayalım:

```bash
# Subnet Associations Oluşturalım
 
```

* Kodun son hali:

```bash
provider "aws" {
    region = "us-east-1"
}
variable vpc_cidr_block{}
variable subnet_cidr_block{}
variable avail_zone {}
variable env_prefix {}

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
        Name: "${var.env_prefix}-subnet-1"
    }
}

# Route Table Oluşturmaca
resource "aws_route_table" "myapp-route-table" {
    vpc_id = aws_vpc.myapp-vpc.id //id mizi tanımladık

    route {
        cidr_block = "0.0.0.0/0" # birinci attribute umuz
        gateway_id = aws_internet_gateway.myapp-igw.id
    }
    tags = {
        Name: "${var.env_prefix}-rtb"
    }
}

# Internet gateway Oluşturmaca
resource "aws_internet_gateway" "myapp-igw" {
    vpc_id = aws_vpc.myapp-vpc.id
    tags = {
        Name: "${var.env_prefix}-igw"
    }
}
# Subnet Associations Oluşturalım
resource "aws_route_table_association" "a-rtb-subnet" {
    subnet_id = aws_subnet.myapp-subnet-1.id
    route_table_id = aws_route_table.myapp-route-table.id
}
```


* Sonrasında yaptığımız değişiklikleri işlemek için Cmd ye terraform apply –auto-approve yazdık.

* 1 resource added yazıyor. AWS VPC Management Route Tables sekmesine F5 yaptıp subnet associations’a gittiğimiz zaman explicit subnet association ımız olduğunu görebiliriz. 

# DEFAUT ROUTE TABLE'I MODIFY ETME

* Peki kendi route table ımızı kullanmaktansa default olanla nasıl devam edebiliriz?--modify ederek-- Şöyle:

- Öncelikle resource "aws_route_table" "myapp-route-table" u ve resource "aws_route_table_association" "a-rtb-subnet" yi comment out yaptık.
- Sonrasında Cmd ye terraform apply –auto-approve yazdık. 
- 2 adet resource u destroy ettik. 
- AWS sayfasını-kaldığın yer zaten- F5 ledik. Gördüğünüz gibi ortadan kalkmış. “internet_gateway” miz hala duruyor. Bunu default route table ile bağlayacağız. Sonrasında;
- Default route table id ye erişmek için cmd ye 
```bash  
  terraform state show aws_vpc.myapp-vpc 
```  
  yazdık. Eriştiğimiz default route table id yi inceledik. 

```bash
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
```

* Cmd ye terraform plan yaz. 
  
* Gördüğünüz gibi resource olarak default route table var. 
  
* Cmd ye terraform apply --auto-approve yaz.

* AWS deki son sayfayı refresh et. dev-main-rtb ye tıkla. Gördüğünüz gibi default trb modify edilmiş-Routes kısmına bakarak daha iyi anlayabilirsin bunu-

* Subnet associations kısmına gel. Main table ın associationunu(ki bu oluşturduğumuz route table hatırlatırım) subnet association olarak aldı. –Subnet without explicit associations kısmında görebiliriz bunu-- İşte bu şekilde default route table ı oluşturabilir & modify edebilirsin. 

* Kodun son hali:

```bash 
provider "aws" {
    region = "us-east-1"
}
variable vpc_cidr_block{}
variable subnet_cidr_block{}
variable avail_zone {}
variable env_prefix {}

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

# Internet gateway Oluşturmaca
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
```

# İnşa ettiğimiz şeyleri browser üzerinden erişeceğimizden dolayı bir de Firewall(8080) oluşturacağız. Server üzerinden ayrıca SSH portu(22) da açacağız. Bu portlar için de bir Security Group u oluşturacağız.


* my_ip isimli bir variable tanımlayalım: 
```bash
variable my_ip{}
```

* terraform.tfvars ın içine my_ip nin değerini set edelim: Comment deki faydalarının yanında ip yi terraform.tfvars ın içine tanımlamanın bir diğer yararı GitHub daki repomuzda ip adresimizi görünür kılmamamız. 
```bash
my_ip = "141.255.5.197/32" # ip adresini whatsmip.org dan aldık.
# Birden fazla veya dynamic ip adresin varsa değişken tanımlayabilirsin üste.
``` 

# Firewall oluşturalım. Browser üzerinden erişim için port 8080, SSH için de port 22 yi oluşturalım.

```bash
# SECURITY
resource "aws_security_group" "myapp-sg" {
    name = "myapp-sg"
    vpc_id = aws_vpc.myapp-vpc.id

    # Firewall Kuralları
    # a) Incoming Traffic Rules: ssh into EC2 & access from browser
    # 1. SSH
    ingress{ # ingress for incoming
        from_port = 22 # ikisi de 22 çünkü aralıktansa tek bir port istiyoruz
        to_port = 22 # yani burada diyoruz ki port 22 den port 22 ye
        protocol = "tcp"
        #hangi ip adreslerini 22 numaralı porta erişmeye yetkili onu tanımlayalım
        cidr_blocks = [var.my_ip]
    }

    # 2. Firewall
    ingress { # ingress for incoming
        from_port = 8080
        to_port = 8080
        protocol = "tcp"
        # hangi ip adresleri 8080 numaralı porta erişmeye yetkili onu tanımla
        cidr_blocks = ["0.0.0.0/0"] # Bunu yazarak tarayıcıdan giren bütün
        #                ip adreslerini port 8080 için erişilebilir kıldık.

    }

    # b) Outgoing Traffic Rules: installations & fetch Docker Image
    egress { # egress for exiting
        # Hiçbir şeyi kısıtlamıyoruz çünkü içerideki bütün trafiğin dışarı çıkmasını istiyoruz.
        from_port = 0
        to_port = 0
        protocol = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
    tags = {
        Name: "${var.env_prefix}-sg" # sg= security group
    }
}
```

* Cmd ye terraform plan, terraform apply –auto-approve yaz. 
  
* Sonrasında AWS VPC deki SecurityGroups kısmına(yanda) tıkla. Dev-sg yazan bizimki diğerleri default. 
  
* Myapp-sg nin üzerine tıkla. Görüldüğü gibi 2 adet port aralığı var: 8080 ve 22. 
  
* Outbound rules a tıkla: o da egress in içindekiler. Security Groups sayfasını geri aç. dev-sg nin altındaki default security grouplardan birini kullanmak istersek napmalıyız?  Burayı

```bash
resource "aws_default_security_group" "default-sg" {
    vpc_id = aws_vpc.myapp-vpc.id
}
```

ve burayı

```bash
tags = {
    Name: "${var.env_prefix}-default-sg"
}
```
güncellemen yeterli.

* Cmd ye terraform plan, terraform apply –auto-approve yaz

* AWS/Security Groups sayfasını yenile. 

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
        Name: "${var.env_prefix}-subnet-1" 
    }
}

# Internet gateway Oluşturmaca
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

# SECURITY
resource "aws_default_security_group" "default-sg" { # DEĞİŞTİ
    vpc_id = aws_vpc.myapp-vpc.id # DEĞİŞTİ


    ingress{ 
        from_port = 22 
        to_port = 22 
        protocol = "tcp"
        cidr_blocks = [var.my_ip]
    }

    # 2. Firewall
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
        Name: "${var.env_prefix}-default-sg" # DEĞİŞTİ
    }
}
```

# Automate AWS Infrastructure

* EC2 instance oluşturmamız için daha neye ihtiyacımız var? Hangi configuration ları yapmalıyız? Veya başka neler yapmalıyız? AWS EC2 instance için bir configuration oluşturalım..
  
* Şu web adresine gir: https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#LaunchInstanceWizard: 
 
- AWS Instance için bir attribute tanımlamalıyız
- 1) buna ilgili web sitesinden seçeceğimiz image in id sini ekleyelim
- AMI değerini hardcode yapmamalıyız çünkü her yeni pakette bu değer değişiyor.
- Bunun yerine AMI değerini dinamik olarak fetch edebiliriz.

```bash
# AMI değerine hardcode etmeden query leyerek ulaşma
data "aws_ami" "latest-amazon-linux-image" {
    most_recent = true # most recent image version
    owners = ["amazon"] # image ın sahibi amazon olsun
    filter {# query in için kriterlerin neler burada belirleyebilirsin
        name = "name"
        values = ["amzn-ami-hvm-*-x86_64-gp2"] # başlangıcı -*- öncesi
                                        # bitişi -*- sonrası olan AMI leri query le.
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
```

* Cmd ye terraform plan yazalım. 
  
* Çıktıdaki name i  Community Amazon AMI nin üst kısmındaki arama yerinde aratalım. Gördüğünüz gibi seçtiğimiz filtreden seçmiş AMI yi. Sonrasında şu kısmı yazalım:

```bash
resource "aws_instance" "myapp-server"{
    ami = data.aws_ami.latest-amazon-linux-image.id
}
```


#  VPC lerin içine de EC2 Instance ları inşaa et

* instance_type isimli bir variable oluştur:

```bash
variable instance_type{}
```

* terraform-dev.tfvars a da değerini gir:

```bash
instance_type = "t2.micro"
```

* main.tf I şu şekilde güncelle:

```bash
resource "aws_instance" "myapp-server"{ # resource u aws_instance olan
                        #myapp-server isimli bir configuration oluşturduk.
# Bu noktada 2 adet instance oluşturmamız gerekiyor:
    ami = data.aws_ami.latest-amazon-linux-image.id # EC2 server'ının dayandığı image
    instance_type = var.instance_type

    
    # Bu kısım opsiyonel, kodu optimize etmek için yapıyoruz.
    subnet_id = aws_subnet.myapp-subnet-1.id
    availability_zone = var.avail_zone
    
    # Yukarıdaki 2 satırda şunu dedik:
    # server'ı subnet_id den al,
    #belirttiğimiz zone a ata

    
    # Bu kısım gerekli
    associate_public_ip_address = true # ip adreslerine browserdan eriş
```

*  AWS den EC2 ya gir. Key pairs e tıkla. Create key pairse tıkla. 
*  Name: server-key-pair, ppk seçili kalsın, Create key pair e tıkla.
*  İndirilen dosyayı .ssh'e taşı--.ssh iniz yoksa ya da hata alıyorsanız direk directory nize de taşıyabilirsiniz--: Elle yap bunu
* Yukarıdaki kod devan ediyor..
```bsh    
    key_name = "mbkey"

    tags = {
      Name: "${var.env_prefix}-server"
    } 
}
```

* Cmd ye terraform plan ve terraform apply –auto-approve yaz. 
  
* AWS dashboard I aç sayfayı yenile(https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:). Gördüğünüz gibi server ımız çalışıyor arkadaşlar-dev-server bizimkisi-Tıkla 
  
*  dev-server a. terraform.tfvars I aç. Gördüğünüz gibi arkadaşlar Private IP adresimiz vpc_cidr_block ve subnet_cidr_block arasında. 
