Herkese merhaba, bu yazıda örnek bir uygulamayı Google CloudBuild kullanarak GKE üzerinde farklı ortamlara deploy etmekten bahsedeceğim.

Yazıda ele alacağımız örnek uygulama Golang ile yazılmış ve bir tane endpointi olan bir rest api. Örnek uygulamayı deploy edeceğimiz ortam ise staging ve production namespaceleri bulunan bir Kubernetes clusterı.

Uygulamamızın deployment senaryosu şu şekilde olacak,

1. Master branchinde yapılan geliştirmeler release branchine merge edildiğinde, uygulama build edilip staging namespaceine deploy edilecek.
2. Her şey yolunda ise production kelimes ile başlayan oluşturulacak tag ile staging namespaceine deploy edilen versiyon production namespaceine de deploy edilecek.


Ben K8s üzerine yapılacak deploymentlarda Helm kullanmayı tercih ediyorum. Bu nedenle projede helm dizini altında çok basit bir chart oluşturdum. Burada  değişkenleri tanımladığım values.yml, values-staging.yml ve values.production.yml olarak 3 adet values dosyam mevcut. values.yml dosyasında uygulamamı deploy edeceğim tüm ortamlar için ortak olacak değerler varken values-staging.yml ve values-production.yml dosyamda ortamlara göre değişkenlik gösterecek değerim bulunuyor.


Projemizde Google CloudBuild Pipeline'ının adımları tanımladığımız cloudbuild.yaml ve cloudbuild-production.yaml adında iki tane dosyamız var. Bu iki dosyamıza karşılık Gcloud Console da bulunan CloudBuild Dashboard ekranında iki tane trigger oluşturacağız.
Bu işlem için önce Cloud Build ekranından sol menüden Triggers'a girdikten sonra Connect Repository butonuna tıklayarak Github Repolarımıza erişim izni vermemiz gerekiyor.


![Alt text](docs/1.jpg?raw=true "Connect Repository")



Github için gerekli izinleri verdikten sonra yine Triggers ekranında Create Trigger butonuna tıklayarak cloudbuild.yaml dosyamız için ilk triggerımızı oluşturacağız.


![Alt text](docs/2.jpg?raw=true "Create CloudBuild Staging Trigger ")


Burada şunları belirtiyoruz,
 - Trigger'ın Adı. Bizim senaryomuzda build-and-deploy-to-staging olacak.
 - Event olarak daha önce bahsettiğim gibi release branchine yapılacak push ile pipeline tetiklenecek. Bunuda Event bölümünde "Push to a branch" seçeneği ve Source bölümünde Branch alanına release yazarak sağlıyoruz.
 - Son olarak Build configuration bölümünde Cloud Build configuration file (yaml or json) seçeneceğini seçip triggerımızın kullanacağı cloudbuild.yaml dosyasını gösteriyoruz.

Triggers ekranında Create Trigger butonuna tıklayarak cloudbuild.yaml dosyamız için ilk triggerımızı oluşturacağız.



![Alt text](docs/3.jpg?raw=true "Create CloudBuild Production Trigger ")


Burada ise  şunları belirtiyoruz,
 - Trigger'ın adı. Bizim senaryomuzda bu kez deploy-to-production olacak.
 - Event olarak ise bu sefer Push new tag seçeneceğini seçiyoruz ve Source bölümündeki Tag kısmına ^production yazıyoruz.
 - Son olarak Build configuration bölümünde "Cloud Build configuration file (yaml or json)" seçeneceğini seçip triggerımızın kullanacağı cloudbuild-production.yaml dosyasını gösteriyoruz.


Bu aşamalarla Google Cloud tarafındaki tanımlamalarımız bitti.


cloudbuild dosyalarımızı inceledeğimizde de temel olarak substitutions ve steps olarak iki bölümden oluşuyor. substitutions kısmında steps alanında çalışacak tasklarda kullanılacak değişkenleri tanımlıyoruz. steps alanında ise taskları tanımlıyoruz.
CloudBuild ile çalıştığımızda dikkat etmemiz gereken bir kaç nokta var. İlk olarak steps alanında tanımlanan tasklar paralel şekilde çalışıyor. Bir taskın bir diğer taskı beklemesi ve sonrasında çalışması için waitFor ifadesinden yararlanıyoruz. Buna ek olarak taskların içinde substitutions alanında tanımlanan değişkenleri kullannmak için $VAR_NAME şeklinde kullanılırken task içindeki yada taskın çalışacağı container içindeki değişkenler $$VAR_NAME şeklinde kullanılıyor. 

