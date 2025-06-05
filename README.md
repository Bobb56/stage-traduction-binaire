# Stage traduction dynamique de binaires x86 vers RISC-V

## I - Les solutions de traduction dynamique de binaires déjà existantes : Box64 et QEMU

Box64 est un traducteur dynamique de binaires x86_64 vers plusieurs autres architectures cibles incluant ARM64, RISCV64. Box64 a besoin pour fonctionner d'être exécuté sous Linux et ne peut traduire que des exécutables Linux. C'est un traducteur axé sur la performance dont l'un des principaux buts est de pouvoir faire tourner à vitesse acceptable des jeux vidéos demandants en ressources.
Box64 est un traducteur dynamique, c'est-à-dire qu'il traduit le code binaire x86_64 en code natif au fur et à mesure de l'exécution. Une fois traduite, une portion de code n'a plus à être traduite de nouveau et pourra être exécutée nativement la prochaine fois que son exécution sera demandée. La traduction se fait par blocs : lorsqu'il s'agit de traduire une nouvelle portion de code, Box64 découpe un bloc de code, puis le traduit, et l'exécute d'une traite. Ensuite soit il saute à un nouveau bloc déjà traduit et l'exécute à son tour, soit il arrive dans une zone non encore traduite, crée un bloc et recommence le processus.
Pour traduire un bloc de code, Box64 décode chaque instruction x86_64 du bloc, et va remplacer chaque instruction par des instructions correspondantes dans le langage machine hôte. L'architecture x86_64 étant particulièrement complexe et riche en instructions, déterminer les instructions équivalentes à chaque instruction est un travail particulièrement long et fastidieux.
De plus, une fois ce traducteur de code écrit pour générer du code ARM64 par exemple, il faut reprendre tout ce travail de zéro pour pouvoir générer du code RISCV64.


QEMU est un émulateur rapide généraliste utilisant un traducteur dynamique binaire portable. Par portable, on entend que QEMU résoud partiellement le problème de la réécriture du traducteur dynamique.
Le premier but de QEMU est de pouvoir exécuter un système d'exploitation non modifié dans un autre système d'exploitation, peu importe l'architecture cible et l'architecture hôte.
En plus de l'émulation complète de la machine, QEMU est capable d'exécuter directement des binaires Linux d'une architecture différente de l'architecture hôte.
Pour régler le problème de la difficulté du portage des émulateurs tels que Box64, QEMU utilise un générateur automatique de générateur de code appelé dyngen. Les générateurs de code générés par dyngen ne transforment pas directement du code binaire de la machine cible vers la machine hôte, mais passe par une représentation intermédiaire des instructions à traduire appelée les micro-operations. 
Ainsi pour émuler une nouvelle architecture avec QEMU, le seul travail est de traduire à la main chaque instruction en micro-opérations. Et contrairement à Box64, une fois ce travail effectué, l'architecture cible peut être émulée sur n'importe quelle autre architecture.
Pour cela, les micro-opérations sont implémentées une et une seule fois dans des petites fonctions C, et compilées dans des fichiers objet grâce à GCC. Ainsi le travail de génération de code spécifique machine est réalisé par GCC. dyngen utilise ensuite ces fichiers objet pour générer le générateur de code.

### Installation de Box64 dans QEMU sur Linux
Dans un premier temps, j'ai cherché à appréhender ces outils afin de comprendre comment les utiliser et aussi comment ils fonctionnent. J'ai donc entrepris de faire fonctionner Box64 dans QEMU.

Tout d'abord, il faut installer QEMU. Comme dit plus haut, QEMU a deux modes de virtualisation : le mode full system emulation et le mode user emulation. Voici les commandes permettant d'installer les deux :
```
apt-get install qemu-system
apt-get install qemu-user-static
```

Ensuite, pour lancer QEMU, les lignes de commandes sont assez complexes. Il y a plusieurs manières de charger un OS avec QEMU. D'abord, on peut lui spécifier une image disque contenant un OS entier, à ce moment là, QEMU démarre ce disque comme un BIOS classique. Dans le cas du RISCV, il est probable qu'il faille spécifier à QEMU quel BIOS utiliser via l'option -bios.
On peut aussi spécifier à QEMU une image disque contenant uniquement le système de fichiers, pas de bootloader, et lui indiquer à côté le kernel à utiliser. C'est vers cette option que je me suis tourné.
Pour faire tourner Linux dans QEMU RISCV, je me suis aidé de ce [site](https://canonical-ubuntu-boards.readthedocs-hosted.com/en/latest/how-to/qemu-riscv/), et j'ai téléchargé l'image disque d'Ubuntu que l'on peut trouver [ici](https://ubuntu.com/download/risc-v).

On nous indique un paquet à installer, qui permet de télécharger le kernel à utiliser, et finalement on n'a plus qu'à lancer la commande indiquée et ça fonctionne (en tous cas pour moi, ce n'est pas exclu que ça ne marche pas pour tout le monde, vu le nombre de tutos que j'ai testés avant que ça fonctionne).

La commande indiquée par la page d'Ubuntu ne permet pas de communiquer avec l'instance QEMU en SSH. Voici donc une petite modification de la commande qui permet d'envoyer/récupérer des fichiers via scp et de se connecter en ssh : 
```
qemu-system-riscv64 \
    -machine virt -nographic -m 16384 -smp 8 \
    -kernel /usr/lib/u-boot/qemu-riscv64_smode/uboot.elf \
    -device virtio-net-device,netdev=usernet -netdev user,id=usernet,hostfwd=tcp::10000-:22 \
    -device virtio-rng-pci \
    -drive file=ubuntu.img,format=raw,if=virtio
```
Cette commande utilise le port 10 000, mais ça n'a pas d'importance, un autre port ferait l'affaire.

L'instance de QEMU que l'on lance en suivant ce tuto est accessible via SSH, et a accès à internet. On peut donc installer des paquets via apt, télécharger du contenu avec wget, etc. D'ailleurs toutes les modifications que l'on effectue dans le système de fichiers sont persistantes, en quittant QEMU et en redémarrant avec la même commande, les fichiers seront toujours là.
Une fois dans Ubuntu, pour quitter (éteindre l'OS), il suffit de lancer cette commande dans le terminal :
```
systemctl poweroff
```

#### D'autres tutos qui ne fonctionnent pas

Comme je l'ai dit, ce n'est qu'après nombre de tentatives infructueuses que je n'ai réussi à faire fonctionner QEMU.
J'ai en premiers lieu essayer de compiler moi-même un noyau Linux, que j'utilisais avec une image disque fabriquée à partir de BusyBox. Ensuite j'ai également testé le Berkeley BootLoader (BBL) avec une image disque de Fedora. Voici d'ailleurs [une page qui explique comment utiliser Fedora RISCV dans QEMU](https://fedorapeople.org/groups/risc-v/disk-images/readme.txt).
On peut télécharger une image disque qui s'appelle stage4-disk.img, et ensuite lancer QEMU avec le BBL et cette image disque.

ce tutoriel n'a pas fonctionné pour moi, en revanche c'est en m'intéressant à [Oxtra]([Oxtra](https://github.com/oxtra/oxtra)) (un traducteur dynamique de binaires x86 vers RISCV, comme Box64), que j'ai pu avoir un QEMU à moitié fonctionnel. En effet Oxtra met à notre disposition un conteneur Docker avec QEMU préinstallé, le BBL et l'image stage4-disk.img à l'intérieur. Il suffit de lancer le conteneur et de démarrer le script shell start.sh, et Fedora se lance dans QEMU. Le seul problème est que, bien qu'internet soit accessible depuis Fedora à l'intérieur de QEMU, les DNS ne fonctionnent pas. C'est donc compliqué d'installer des commandes, des applications. En revanche on peut toujours envoyer des fichiers via scp, à condition d'ouvrir le port 10000 sur le conteneur. On peut pour ça lancer le conteneur avec cette commande :
```
sudo docker run -it --rm -p 10000:10000 plainerman/qemuriscv:fedora
```

Une fois dans le Fedora dans QEMU dans le conteneur Docker, j'ai tout de même essayer d'installer Box64. Il y a deux manières de faire fonctionner Box64 dans QEMU. La première est de cloner le repo Github directement dans QEMU, et de lancer la compilation dans QEMU. Cette technique a quelques inconvénients :
- Quand les DNS ne fonctionnent pas, c'est compliqué de cloner un repo Github. Ce souci peut quand même être réglé en envoyant le contenu du repo compressé via scp, et en le décompressant.
- Ensuite, pour compiler Box64, il y a besoin de la commande CMAKE, pour fabriquer les Makefiles. Or comme il n'y a pas de DNS fonctionnel, on ne peut pas installer CMAKE. Une solution a été de fabriquer les Makefile avant d'envoyer le contenu du github avec scp. Mais ça ne fonctionne pas non plus puisque l'étape de compilation (```make```) a elle aussi besoin de cmake pour fonctionner.
- Quand bien même on arrivait à lancer la compilation de Box64, celle-ci serait très lente car elle se ferait à travers la couche de virtualisation de QEMU.

La deuxième manière d'utiliser Box64 est de le compiler sur un OS x86, avec la toolchain RISCV64. Cette toolchain est indispensable quand on veut développer sur RISCV, et elle peut être installée via apt. J'ai aussi essayé de la compiler, cela dure des heures et des heures et ne fonctionne même pas au bout du compte car il y a toujours des problèmes de dépendances, des commandes soi-disant non installées alors que si, des paquets dans une version trop ancienne mais pour lesquels il n'esxiste pas de version plus récente, des paquets qu'on ne peut pas installer via apt, qu'il faut compiler soi-même, mais dont la compilation a d'autres dépendances, qu'il faut aussi compiler, ... C'est mieux de pouvoir installer la toolchain grâce à apt.

Une fois que l'on a installé la toolchain, il suffit de spécifier une option de compilation de Box64 pour qu'il utilise le compilateur riscv64 au lieu de gcc classique, et le tour est joué.

Ensuite, il ne reste plus qu'à envoyer l'exécutable généré (un exécutable RISCV donc) via scp à QEMU ainsi que les librairies dynamiques de Box64, et on peut le lancer.
C'est sans compter sur le fait que l'image disque stage4-disk.img date de 2018 et qui n'a donc pas les bonnes versions de la libc standard.

Je mets d'ailleurs [le lien](https://fedorapeople.org/groups/risc-v/disk-images/) pour télécharger le BBL et stage4-disk.img mis à disposition par Fedora.

Donc impossible de faire fonctionner Box64 dans le conteneur Docker.

En revanche, une fois que j'ai réussi à faire fonctionner QEMU avec l'image d'Ubuntu, tout fonctionnait parfaitement bien.
J'ai tenté de coompiler Box64 directement dans QEMU, mais le compilateur a crashé parce qu'il n'y avait pas assez de RAM. J'en ai donc profité pour augmenter la taille du disque, la quantité de RAM, et le nombre de coeurs de la machine émulée par QEMU.
Pour agrandir le disque, il suffit de lancer une commande de ce style :
```
qemu-img resize -f raw ubuntu-24.04-preinstalled-server-riscv64.img +5G
```

Ensuite, j'ai directement envoyé dans QEMU l'exécutable compilé sur mon PC avec la toolchain RISCV64, les librairies dynamiques, et Box64 a parfaitement bien fonctionné.
Je précise que la commande indiquée sur (cette page)[] pour générer les Makefile avant de compiler Box64 va lancer une compilation de Box64 en mode interpréteur. Il est possible d'activer le recompilateur dynamique (dynarec) qui accélère drastiquement l'exécution par rapport à l'interpréteur en ajoutant l'option ```-D RV64_DYNAREC``` à cette commande :
```
cmake .. -D RV64=1 -D CMAKE_BUILD_TYPE=RelWithDebInfo -D CMAKE_C_COMPILER=riscv64-linux-gnu-gcc -D USE_CCACHE=ON
```

#### Tests de performance de Box64 vs QEMU natif vs natif
