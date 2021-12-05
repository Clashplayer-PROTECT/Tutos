## Créaction TUNNEL OVH à la Maison avec IP FAILOVER grâce à WireGuard



```
Ce upgrade de la documentation de MichelBaie en raison que le tutoriel n'est plus du tout à jour.

```

J'ai un serveur chez moi, et j'ai besoin d'IP...

Grâce à mes **IP Failover**, j'évite de donner celle de ma box qui est vulnérable, et je donne celle **d'OVH** [protégée par leur petite protection DDOS](https://www.ovh.com/fr/anti-ddos/technologie-anti-ddos.xml).

Si vous souhaitez faire la même chose que moi, vous êtes au bon endroit.

C'est compatible Windows, Linux et tout autre étant donné que [WireGuard](https://www.wireguard.com) est un [VPN d'Avenir](https://www.google.com/search?q=wireguard&tbm=nws) qui est désormais inclut dans beaucoup de kernels.

### 1 - Commençons par commander un VPS OVH

Pourquoi j'ai choisi **OVH** ?

Le triste gros avantage d'OVH c'est que c'est (à ma connaissance) le seul hébergeur français à proposer l'achat d'**IP Failover à** **2€50 à vie** (tant que le service reste actif). Cela va nous permettre de réaliser de **sérieuses économies** au bout d'un an.

- L'avantage d'un [VPS](https://www.ovhcloud.com/fr/vps/compare/) c'est que les offres commencent directement à partir de **débits supérieurs à 100MB/S** : Si vous faites tourner un **RDP Windows** les téléchargements seront **limités** à ce débit. Cependant, la qualité du réseau sera tout aussi stable.
- Voici un petit tableau des prix si vous avez besoin de comparer rapidement :

| VPS START     | VPS Value        | VPS Essential     | VPS Confort       |
| ------------- | ---------------- | ----------------- | ----------------- |
| 100 MB/S      | 250 MB/S         | 500 MB/S          | 1 GB/S            |
| 3€ (par mois) | 5€ HT (par mois) | 10€ HT (par mois) | 20€ HT (par mois) |

Je précise qu'on a **pas besoin de gros** **CPU** ou **RAM**, seulement de réseau car WireGuard est **très léger**.
J'ai **personnellement choisi le tout premier VPS START** : *s1-2*

### 2 - Installons notre VPS

Dans ce tutoriel, je vais utiliser **Debian 11**, cela peut changer certaines choses comme le login SSH ou autre, mais prenez le même que moi au moins on sera sûr d'avoir des choses identiques.
⚠ **Utilisez Debian 11pour être sûr d'être 100% compatible : Le tuto peut ne pas fonctionner ou bien manquer de repos sur d'autres distrib.** ⚠ 

Voici une petite liste des trucs à faire après avoir reçu notre service :

- Installer WireGuard et ses dépendances
- Configuration de notre IP FAILOVER 
- Créons notre tout premier profil WireGuard
- Modification du réseau DEBIAN


Voici les commandes à exécuter pour tout préparer et lancer notre installation :

```
sudo su
apt install curl bash sudo wget resolvconf wireguard-tools wireguard -y
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
chmod +x wireguard-install.sh
./wireguard-install.sh
```

Une fois ceci fait il va lancer notre petit setup interactif :

```bash
Welcome to the WireGuard installer!
The git repository is available at: https://github.com/angristan/wireguard-install

I need to ask you a few questions before starting the setup.
You can leave the default options and just press enter if you are ok with them.

IPv4 or IPv6 public address: <ipvps>
```

Au dessus il demande l'IP publique du VPS, par défaut il devrais bien la détecter donc laissons-là.

```bash
Public interface: eth0
```

On laisse l'interface réseau d'écoute par défaut.

```bash
WireGuard interface name: wg0
```

Toujours laisser l'interface WireGuard par défaut.

```
Server's WireGuard IPv4: 10.66.66.1
```

Ici, il nous demande lequel sera notre réseau virtuel WireGuard. Il faut que ce réseau **ne sois pas utilisé dans votre LAN**, donc ne remplissez pas par votre réseau local, laissez les valeurs par défaut qui marcheront très bien.
Pour l'IPV6 oublions, OVH ne commercialise pas encore d'IPV6 Failover 🤔

```
Server's WireGuard port [1-65535]: XXXXXX
```

Ici, il nous demande le port d'écoute de WireGuard, **notez-le**, on en aura besoin pour plus tard.

```
First DNS resolver to use for the clients: 176.103.130.130
Second DNS resolver to use for the clients (optional): 176.103.130.130
```

Utilisons les meilleurs DNS (chacun son avis) : ceux de [CloudFlare](https://1.1.1.1/) : `1.1.1.1, 1.0.0.1`

Une fois tout ce QCM de remplis, il va tout préparer et nous demandera ensuite le nom de notre tout premier client.

```
Client name: MaVM
```

Ici, pour lui donner un nom facile à reconnaitre je vais l'appeler **MaVM**, mais vous pouvez l'appeler comme vous souhaitez.

```
Client's WireGuard IPv4: 10.66.66.2
```

Ici il nous demande son IP sur le LAN, donc laissez par défaut et tout se passera bien. Mais **notez-le** quand même.


Ouvrons-le (`/etc/wireguard/wg0.conf`), vous devriez avoir la même configuration en changeant les arguments

```bash
[Interface]
Address = 10.66.66.1/24,fd42:42:42::1/64
ListenPort = <PortWireGuard>
PrivateKey = 8HNFm5fD7tE7kPCeJ7PUrRo7N/dhw1X1FPS3n79nZkw=
PostUp =  ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
PostDown =  ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o ens3 -j MASQUERADE


[Peer]
PublicKey = qLpTabk+liUFUPoDklUwr5LKAQSOLdjsUrvCnWeEG1E=
PresharedKey = iLMJy9h5eg8xE5Ou1+BS6hZ716cifCyWD20Dj0MZoRs=
AllowedIPs = <IPFailover>/32,10.66.66.2/32,fd42:42:42::2/128


```

Je vous explique ce que signifie chacun des arguments <> à remplacer :

* `<PortWireGuard>` : Le port WireGuard que je vous ai demandé de noter au dessus.

* `<IPFailover>`/32 : Il s'agit de l'IP OVH que vous venez d'acheter.

  On sauvegarde notre fichier avec CTRL X + Y + Entrée

Puis sauvegardez le fichier. Et on relance WireGuard : 

```
systemctl stop wg-quick@wg0
systemctl start wg-quick@wg0
```

Une fois les commandes terminées, il est possible que quelques erreurs soient survenues et est préférable de redémarrer le VPS.

#### Nettoyer nos clients Wireguard

Par défaut le script n'est pas vraiment optimisé à cet usage. Nous allons donc prendre le client qu'il nous a gentiment générer et le pimper.
Nous allons retirer l'IPV6 et changer la gateway VPN

Celui-ci devrait être situé dans le dossier /root (par défaut quand on est root), il vous suffit donc simplement de faire la commande `ls`  et voici notre premier profil :

```bash
root@s1-2-gra7:~# ls
wg0-client-MaVM.conf  wireguard-install.sh
root@s1-2-gra7:~#
```

ici mon client est `wg0-client-MaVM.conf` mais vous devriez avoir un autre que vous avez définit juste avant.

On va juste débroussailler quelques lignes dans celui-ci afin d'avoir un truc très propre.
Ouvrons-le (`nano monprofil.conf`)

```
[Interface]
PrivateKey = 8HNFm5fD7tE7kPCeJ7PUrRo7N/dhw1X1FPS3n79nZkw=
Address = 10.66.66.2/32,fd42:42:42::2/128
DNS = 1.1.1.1,1.0.0.1

[Peer]
PublicKey = qLpTabk+liUFUPoDklUwr5LKAQSOLdjsUrvCnWeEG1E=
PresharedKey = iLMJy9h5eg8xE5Ou1+BS6hZ716cifCyWD20Dj0MZoRs=
Endpoint = <IPVPS>:59622
AllowedIPs = 0.0.0.0/0,::/0
```



* `<IPVPS>` : IP de notre VPS qui sert de Gateway.

  

Vous pouvez copier coller le contenu du fichier dans un éditeur de texte comme visual studio code et l'enregistrer sous .conf

**Modification du réseau debian**

Faisons quelques commandes

```
echo 1 >> /proc/sys/net/ipv4/conf/all/proxy_arp
nano /etc/sysctl.conf
```

Ajoutons dans le fichier 

```
net.ipv4.ip_forward=1
net.ipv4.conf.all.proxy_arp=1
```

On sauvegarde notre fichier avec CTRL X + Y + Entrée

### Installer Wireguard sur notre VM Client

Wireguard a la particularité d'être multi-os, multi-plateformes, partout. Android, Windows, Linux, Router, Télé, Par-tout.

Vous pouvez [regarder leur documentation](https://www.wireguard.com/install/) qui vous expliquera très simplement comment installer wireguard.
Mais grosso-modo, si vous avez un os très compatible (Debian 10, Ubuntu 18.04/20.04) il vous suffit de faire la commande

`apt install wireguard wireguard-tools  resolvconf `

Une fois celle-ci faite on a besoin d'installer notre profil. Je vous conseille de vous connecter en ssh et ducoup de ne plus utiliser noVNC (Proxmox) car on ne peut pas copier coller ce qui va nous être très utile. (Pour obtenir l'ip locale : `ip a` )

En ssh, faisons la commande `nano /etc/wireguard/wg0.conf` puis collons notre profil que nous avions pimpé juste avant dans le terminal. Sauvegardons le fichier et voila !

Puis effectuez la commande :

```bash
systemctl start wg-quick@wg0
systemctl enable wg-quick@wg0
```

Normalement si vous faites un `curl ifconfig.me` vous devriez voir l'ip failover OVH. Et voilà 😆 on a une première ip OVH sur une VM dans sa maison.

Par défaut avec la commande enable, le service wireguard montera et démarrera au démarrage afin d'avoir aucune commande supplémentaire pour monter notre interface.

### Ajouter des autres IP supplémentaires

C'est bon ! Vous avez tout en main pour déployer des ips OVH. Néanmoins je vais quand même vous rappeler les commandes nécessaires pour déployer d'autres profils.

Déjà n'oubliez pas d'être root, car tout se passe en root pour la commande `bash wireguard-install.sh`

```bash
It looks like WireGuard is already installed.

What do you want to do?
   1) Add a new user
   2) Revoke existing user
   3) Uninstall WireGuard
   4) Exit
Select an option [1-4]:
```

Sélectionnez 1 afin de déployer un nouveau profil.

Dans le setup il vous demandera a un moment l'ip locale du client, il faut mettre une ip qui n'est Ducoup pas utilisée. Tout à l'heure j'ai pris 10.66.66.2, il faudra donc mettre 10.66.66.3 etc ...

Une fois le setup terminé, le profil sera disponible dans le dossier /root



### Merci

Merci d'avoir suivi ce tutoriel, espère que celui vous aura été utile (pour ma part sa m'a changé ma vie, j'ai déjà 16 ip ovh chez moi 😆, la limite :().
Le but de mon site c'est tout ce tutoriel : des trucs utiles qui peuvent servir a tout le monde.

Merci à [Mael](https://github.com/maelmagnien) d'avoir Patch certains bugs dans le tuto.

