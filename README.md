# WordPress-i-OpenShift-Grupp4
Grupparbete Containerteknologi, WordPress i OpenShift, DevOps 24 grupp 4

Målbild
Publik URL: WordPress nås via en OpenShift Route.
Beständighet: Innehåll överlever pod-omstarter genom PVC:er (en för WP, en för DB).
Säker körning i OpenShift: Container-bilderna kör som icke-root och tål OpenShifts slumpmässiga UID:er.

Arkitektur (översikt)
[Internet] ──> [OpenShift Router/Route] ──> [Service: blog-wordpress]
                                         └─> [Pod: WordPress]
                                                   └─ PVC: wp-data
                                         └─> [Pod: MariaDB]
                                                   └─ PVC: db-data

Varför dessa val?
Bitnami WordPress Helm chart
  Motivering: Bitnami-bilderna är byggda för att köras som icke-root/arbitrary UID — exakt vad OpenShift kräver. Chartet skapar färdigt WordPress + MariaDB, PVC:er och probes. Det minskar handpåläggning och felkällor               jämfört med att skriva alla YAML:er själv.
MariaDB som separat pod
  Motivering: Tydlig separation mellan app och databas underlättar uppgraderingar och felsökning. PVC på databasen ger beständighet.
OpenShift Route framför NodePort
  Motivering: Route är first-class i OpenShift, ger er publik DNS-vänlig åtkomst utan att peta i noder eller brandväggar.
PVC med standard StorageClass
  Motivering: Enkla dynamiska volymer som följer klustrets praxis. Storlek kan senare skalas upp.
Secrets & ConfigMaps via Helm-värden
  Motivering: Lösenord och config injiceras som K8s-objekt, versioneras via Helm values utan att hamna i manifest direkt.


Dag 1.
Stötte först på problem i GUI med att helm inte fungerade (som visade sig vara en känd bug) så valde då att försöka importera direkt från YAML och tog då fram den fil som är döpt "wordpress.yaml" men stötte då på lite olika problem som att det inte gick att binda till någon PVC. Testade både lvms-vg1 och nfs utan framgång

Dag 2.
Vi radera allt inventory i projektet och installerade Command Line Tool oc direkt så att vi kunde gå via CLI och då gick det lite bättre, men fortfarande några problem.
Vi testade först med StorageClass nfs men PVC fastnade i pending.
Ändrade i wordpress.yaml till lvms-vg1 och då blev statusen äntligen: bonded
Däremot kunde WordPress inte connecta med Mariadb
Efter en längre felsökning utan framgång bestämde vi oss för att ge helm en sista chans så vi raderade hela Inventory igen.
Installerade helm, med lite problem där helm inte först hittades i env:$PATH (kör via PowerShell) då den var döpt helm-windows-amd64.exe och behövdes döpas om till helm.exe
När det äntligen var löst körde vi bara kommandot:
helm install wordpress bitnami/wordpress 
  --set service.type=ClusterIP 
  --set route.enabled=true 
  --set wordpressUsername=admin 
  --set wordpressPassword=Admin123! 
  --set mariadb.auth.rootPassword=Admin123! 
  --set persistence.storageClass=lvms-vg1 `
  --set mariadb.primary.persistence.storageClass=lvms-vg1

Allt installerade så fint men det gick inte att komma åt från webläsare.
Med: 'oc get route' insåg vi att vi hade missat att lägga in: --set route.enabled=true
men vi valde bara då på plats att köra: oc expose service wordpress

När nu allt var klart och fungerar kör vi kommandot: oc get deployment -o yaml
och får ut yaml-filen som helm genererade och jag har här valt att döpa den till helm.yaml för att du lätt ska kunna sära på dem
