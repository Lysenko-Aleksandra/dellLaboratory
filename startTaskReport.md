#  Стартовое задание: установка CSI-baremetal на kind #
## Отчёт подготовлен Лысенко Александрой ##
 
В самом начале стоит отметить, что установка CSI-baremetal на kind - непростая задача, если вы не выполнили все условия для неё(go не менее 1.16) или у вас недостаточно памяти для продолжения сборки.Мне понадобилось порядка 40 ГБ.
Установка производилась на виртуальной машине под Ubuntu.

Прежде чем приступить к непосредственной установке необходимо скачать go,  docker и kind, а также завест аккаунт на dockerhub.

##Скачивание go##
переходим в домашнюю директорию
cd ~
качаем тар-архив
!!!!версия не ниже 1.16
curl -O https://dl.google.com/go/go1.18.linux-amd64.tar.gz
извлекаем загруженный архив и установливаем его в желаемую часть системы. Лучше всего хранить его в директории /usr/local
sudo tar -xvf go1.18.linux-amd64.tar.gz -C /usr/local
go должна располагаться внутри директории /usr/local. Рекурсивно изменяем владельца и группу этой директории на root
sudo chown -R root:root /usr/local/go

###Установка переменных окружения###
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin
export PATH=$PATH:$GOPATH/bin:/usr/local/go/bin

##Скачивание doсker##
обновляем apt и качаем пакеты
sudo apt-get update
udo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    
позволяем не-root пользователям пользовальтя командами docker
sudo chmod 666 /var/run/docker.sock

##Скачивание kind##

###Установка переменной окружения###
