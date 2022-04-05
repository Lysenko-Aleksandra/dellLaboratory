#  Стартовое задание: установка CSI-baremetal на kind #
## Отчёт подготовлен Лысенко Александрой ##
 
В самом начале стоит отметить, что установка CSI-baremetal на kind - непростая задача, если вы не выполнили все условия для неё(go не менее 1.16) или у вас недостаточно памяти для продолжения сборки.Мне понадобилось порядка 40 ГБ.
Установка производилась на виртуальной машине под Ubuntu.

Прежде чем приступить к непосредственной установке необходимо скачать go,  docker и kind, а также завест аккаунт на dockerhub.

## Скачивание go ##
# переходим в домашнюю директорию
 «cd ~»
# качаем тар-архив
# !!!!версия не ниже 1.16
curl -O https://dl.google.com/go/go1.18.linux-amd64.tar.gz
# извлекаем загруженный архив и установливаем его в желаемую часть системы. Лучше всего хранить его в директории /usr/local
sudo tar -xvf go1.18.linux-amd64.tar.gz -C /usr/local
# go должна располагаться внутри директории /usr/local. Рекурсивно изменяем владельца и группу этой директории на root
sudo chown -R root:root /usr/local/go

### Установка переменных окружения ###
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin
export PATH=$PATH:$GOPATH/bin:/usr/local/go/bin

## Скачивание doсker ##
# обновляем apt и качаем пакеты
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
# добавляем официальный GPG ключа докера   
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# устанавливаем стабильную версию репозитория
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
 # устанавливаем последнюю версию версию 
sudo apt-get install docker-ce docker-ce-cli containerd.io    
# убеждаемся, что движок докера установлен и работает корректно, запуская image hello-world
sudo docker run hello-world
# позволяем не-root пользователям пользователя командами docker
sudo chmod 666 /var/run/docker.sock

## Скачивание kind ##
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.12.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /<полная директория хранения kind>/kind
### Установка переменной окружения ###
export PATH=$PATH:<полная директория хранения kind>

## Скачивание источников csi-baremetal и csi-baremetal-operator ##
git clone https://github.com/dell/csi-baremetal-operator
git clone https://github.com/dell/csi-baremetal

Далее можно смело следовать гайду https://github.com/dell/csi-baremetal/blob/master/docs/CONTRIBUTING.md

Ниже преведена та же последовательность действий с переводом комментариев, а так же некоторым исправлением
### Сборка ###
## Установка некоторый полезных далее переменных ##
export REGISTRY=<логин_вашего_аккаунта_на_dockerhub>
export CSI_BAREMETAL_DIR=<полная_директория_csi_baremetal>
export CSI_BAREMETAL_OPERATOR_DIR=<полная_директория_csi_baremetal_operator>

## Сборка csi-baremetal ##
cd ${CSI_BAREMETAL_DIR}
export CSI_VERSION=`make version`

# Получаем зависимости
make dependency

# Компилирует прото файлы
make install-compile-proto

# Генерируем CRD
make install-controller-gen
make generate-deepcopy

# Запускаем юнит тесты
make test

# Запускаем sanity tests
make test-sanity

# Очищаем предыдущие артефакты
make clean

# Собираем бинарный файл
make build
make DRIVE_MANAGER_TYPE=loopbackmgr build-drivemgr

# Собираем docker images
make download-grpc-health-probe
make images REGISTRY=${REGISTRY}
make DRIVE_MANAGER_TYPE=loopbackmgr image-drivemgr REGISTRY=${REGISTRY}

## Сборка csi-baremetal-operator ##
cd ${CSI_BAREMETAL_OPERATOR_DIR}
export CSI_OPERATOR_VERSION=`make version`

# Запускаем юнит тесты
make test

# Собираем docker images
make docker-build REGISTRY=${REGISTRY}

## Подготавливаем kind кластер ##
cd ${CSI_BAREMETAL_DIR}

## Если вы проверяли установку kind путём создания кластера, его необходимо удалить
kind delete claster

# Собираем кастомный бинарный файл kind 
make kind-build

# Проверяем кластер
kubectl cluster-info --context kind-kind

# Подготавливаем sidecars 
make deps-docker-pull
make deps-docker-tag

# Ретэгаем CSI images и загружаем их в kind
make kind-tag-images TAG=${CSI_VERSION} REGISTRY=${REGISTRY}
make kind-load-images TAG=${CSI_VERSION} REGISTRY=${REGISTRY}
make load-operator-image OPERATOR_VERSION=${CSI_OPERATOR_VERSION} REGISTRY=${REGISTRY}
Install on kind
cd ${CSI_BAREMETAL_OPERATOR_DIR}

# Устанавливаем оператор
helm install csi-baremetal-operator ./charts/csi-baremetal-operator/ \
    --set image.tag=${CSI_OPERATOR_VERSION} \
    --set image.pullPolicy=IfNotPresent

# Устанавливаем развёртывание
helm install csi-baremetal ./charts/csi-baremetal-deployment/ \
    --set image.tag=${CSI_VERSION} \
    --set image.pullPolicy=IfNotPresent \
    --set scheduler.patcher.enable=true \
    --set driver.drivemgr.type=loopbackmgr \
    --set driver.drivemgr.deployConfig=true \
    --set driver.log.level=debug \
    --set scheduler.log.level=debug \
    --set nodeController.log.level=debug

## Валидация ##
В файле tests/app/nginx.yaml меняем количество реплик на 2 или 3
# Запускаем тестирующее приложение
cd ${CSI_BAREMETAL_DIR}
kubectl apply -f tests/app/nginx.yaml

# Проверяем, что все поды Running и Ready
kubectl get pods

# И все PVCs  Bound
kubectl get pvc
