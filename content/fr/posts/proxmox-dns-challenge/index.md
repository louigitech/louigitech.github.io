+++
title = "Proxmox : Certificat Let's Encrypt avec challenge DNS"
date = 2024-10-23T14:09:43+02:00
description = "Oui"
draft = true
+++


## 📢 Introduction

Que ce soit dans votre homelab ou dans un environnement de production, vous êtes forcément tomber sur ce message à la première connexion de votre hôte Proxmox :

![Self signed alert](img/self_signed_alert.jpg)

Par défaut, l'interface web de Proxmox est livrée avec un certificat auto-signé. Heureusement pour nous, les dernières versions de Proxmox supportent nativement les challenges ACME DNS ! 

Dans cet article, je vais vous expliquer de manière détaillée, toutes les étapes nécessaires à la mise en place d'un certificat Let's Encrypt avec un challenge DNS.

Il faut savoir que la procédure a été réalisation avec un token d'API Infomaniak puisque mon domaine est chez eux. Toutefois, la procédure devrait fonctionner pour les autres fournisseurs, il vous suffira d'adapter les paramètres en fonction. 

### 🔍 Prérequis

3 choses :

✅ Un enregistrement de type A sur votre / vos DNS locaux → Le mien pointe vers minastirith.gattabru.si \
✅ Nom de domaine public → Dans mon cas `gattabru.si`\
✅ Registrar compatible → Domaine chez Infomaniak

### ➕ Avantages
- Pas besoin d'exposer les ports 80 et 443 sur Internet
- Pas besoin d'enregistrement A ou CNAME qui pointe vers votre adresse IP publique
- Par conséquent, renforcement de la sécurité

### ➖ Incovénients
- Nom de domaine public **obligatoire**
- Votre registrar doit être compatible avec le challenge DNS ([la liste ici](https://go-acme.github.io/lego/dns/))
- Complexité de mise en place (et encore, une fois que vous avez compris c'est assez simple)



## ⚙️ Mise en place

### 🛠️ Comptes ACME

On va commencer par déclarer non pas un mais deux comptes ACME sur notre hôte, voici pourquoi :
- Un compte de `staging` sur le directory `Let's Encrypt V2 Staging` qui nous servira à valider que notre configuration ACME est correcte. En cas de problème on pourra débugger et corriger nos erreurs, sans atteindre la limite décrite plus bas et donc, de se retrouver bloqué.

- L'autre compte de `produduction` sur le directory `Let's Encrypt V2`. Une fois la configuration validée, on baculera sur ce compte pour obtenir notre certificat final.

{{< alert >}}
La limite est de 5 échecs par compte, par nom d'hôte, par heure sur le directory `Let's Encrypt V2`.
{{< /alert >}}

Pour créer les premier compte sur le directory `Let's Encrypt V2 Staging`, rendez-vous dans les menus suivants :
1. Datacenter
2. ACME
3. Appuyez sur le bouton `Add`, afin d'ajouter un nouveau compte
4. Renseignez les infos et choisissez bien le directory `Let's Encrypt V2 Staging`. N'oubliez pas également d'accepter les conditions de service (TOS).

![Adding ACME stag. account](img/add_acme_account_stag.jpg)

Vous pouvez répéter les étapes 3 et 4 pour le directory `Let's Encrypt V2` :
![Adding ACME prod. account](img/add_acme_account_prod.jpg)

Nos deux comptes sont créés, nous allons désormais créer un token sur l'API de notre registrar pour le fonctionnement de notre challenge DNS.

### 🪙 Token API (Infomaniak)

Cette action va vraiment dépendre du fournisseur de votre nom de domaine. Le mien étant chez Infomaniak, je vais vous montrer comment faire sur leur plateforme. En général, quelques recherches Google et vous avez votre réponse :
- [Gandi](https://api.gandi.net/docs/authentication/) 
- [OVH](https://help.ovhcloud.com/csm/en-api-getting-started-ovhcloud-api?id=kb_article_view&sysparm_article=KB0042777)

Si comme moi, votre domaine est chez Infomaniak, connectez-vous sur votre manager et créez un nouveau token. Rendez-vous en haut à gauche dans la section `Utilisateurs et profil` :
![Infomaniak API token](img/infomaniak_api_token1.jpg)

Choisissez ensuite `Mon profil` > `Développeur` > puis finalement `Tokens API` pour arriver sur la console de gestion de vos tokens.
Sur cette console, on va créer un nouveau token :

![Infomaniak API token](img/infomaniak_api_token2.jpg)

Renseignez les diverses informations puis créez votre token. 
Limitez bien le scope de votre token uniquement aux `domain - Produits Noms de domaine`. Je vous déconseille vivement de mettre le scope sur "Tous", pour des questions évidentes de sécurité. Pensez aux conséquences si votre token viendrait à être compromis.

![Infomaniak API token](img/infomaniak_api_token3.jpg)

Votre token est désormais créé :

![Infomaniak API token](img/infomaniak_api_token4.jpg)

{{< alert >}}
Une fois votre token créé, copiez-le et stockez le dans un endroit sûr tel qu'un gestionnaire de mot de passe. 
{{< /alert >}}

Comme indiqué, une fois cette page fermée, il ne sera plus possible de voir le token. Dans le cas ou vous auriez oublié de le copier, vous pouvez toujours supprimer celui-ci et en générer un nouveau.

Voila, notre token est créé et il est limité uniquement au scope dont nous avons besoin, on peut retouner sur notre hôte Proxmox pour la suite de la configuration.

### 🔧 Challenge plugin

Toujours dans le même menu `ACME`, dans la section `Challenge Plugin`, appuyez sur `Add` pour ajouter un plugin :
![Adding ACME dns plugin](img/add_acme_dns_plugin.jpg)

Quelques explications :
- `Plugin ID` est le nom que vous souhaitez donner à votre plugin. Je vous conseil de garder un nom cohérent, sans espaces.

- `Validation Delay` est le temps en secondes entre la création de votre enregistrement DNS via l'API et le moment où le fournisseur d'ACME est invité à le valider. Je vous conseil de laisser ce paramètre par défaut et jouer avec plus tard, dans le cas ou ce délais serait trop court pour la validation de votre enregistrement.

- `DNS API` : Qui est votre registrar. Selon votre choix, la section du dessous ne vous demandera pas les mêmes infos. *Exemple avec Cloudflare :*
![ACME dns Cloudflare](img/acme_dns_cloudflare.jpg "On peut clairement voir que nous n'avons pas les mêmes paramètres qu'Infomaniak")

- `API DATA` : Vos paramètres API tels que votre token, le TTL de votre enregistrement ...

{{< alert >}}
Les paramètres `API DATA` dépendent de votre registrar et ne sont pas les mêmes d'un registrar à un autre.
{{< /alert >}}

Mon registrar est Infomaniak, je vais donc consulter la documentation directement sur le [site de go-acme.github.io](https://go-acme.github.io/lego/dns/infomaniak/index.html) :
![ACME Infomaniak config](img/acme_infomaniak_config.jpg)

Les paramètres renseignés sous `Credentials` sont obligatoires. Les autres sous `Additional Configuration`, ne le sont pas mais sont très pratiques afin d'obtenir la validation (on verra ça un peu plus bas).

{{< alert >}}
**ATTENTION :** Si votre domaine est chez Infomaniak, le paramètre `INFOMANIAK_ACCESS_TOKEN` n'est pas bon sous Proxmox... 

Il faut le remplacer par le paramètre nommé `INFOMANIAK_API_TOKEN` (comme la photo ci-dessous) sinon, vous allez finir avec [l'erreur décrite en bas d'article](#-infomaniak-api-token).
{{< /alert >}}

On connait donc l'ensemble de nos paramètres, on peut donc les renseigner dans le champ `API DATA` sur notre hôte Proxmox :

![ACME Infomaniak good config](img/acme_dns_plugin_good_config.jpg)

Comme vous pouvez le constater, j'ai bien remplacé le paramètre `INFOMANIAK_ACCESS_TOKEN` par `INFOMANIAK_API_TOKEN`. Nul besoin de mettre votre token entre guillemets ou apostrophes.

Je rajoute également les paramètres `INFOMANIAK_PROPAGATION_TIMEOUT` et `INFOMANIAK_TTL`. Ces deux valeurs sont à 300 (secondes) afin d'indiquer que la validation s'arrêtera au bout de 5 minutes si elle échoue, et le Time To Live (TTL) de notre enregistrement sera également de 5 minutes.

La configuration de notre challenge DNS est terminée, on peut désormais générer nos certificats !

### 📜 Génération des certificats

Si vous êtes à l'aise avec le challenge DNS, vous pouvez directement passé à la [génération du certificat de production](#-certificat-de-production).

Sinon, je vous propose de faire vos tests avec le compte de **staging**, afin de ne pas atteindre [les limitations citées plus haut](#-comptes-acme).\
Une fois notre configuration de staging validée, on basculera sur le compte de **production**.

#### 🧪 Certificat de staging

Pour cela, rendez-vous dans l'onglet `Certificates` de votre hôte (`Nom de votre hôte` > `System` > `Certificates`).\
Puis basculez le compte ACME sur celui de Staging :

![Proxmox Certificates ACME account](img/proxmox_certificates_acme_account.jpg)

On peut désormais créer un nouveau certificat via le bouton `Add` de cette même page :

![Proxmox Certificates ACME Staging](img/proxmox_certificates_acme_staging.jpg)

1. On vient bien lui spécifier un `Challenge Type` DNS, venir sélectionner notre plugin précédement créé puis on renseigne le nom de notre enregistrement DNS.

2. Une fois les champs remplis, on vient finaliser la création de notre certificat avec le bouton `Create` puis on lance la commande de celui-ci avec le bouton `Order Certificates Now`.

Si tout se déroule bien, vous devriez avoir le résultat suivant :

![Proxmox ACME Staging Success](img/proxmox_certificates_staging_success.jpg)

L'interface web Proxmox va alors se recharger avec le nouveau certificat que vous venez de générer :

{{< alert >}}
Pour le moment, il est encore normal d'avoir un message d'alerte. Le directory `Let's Encrypt V2 Staging` ne génère pas de certificats issus d'une authorité de certification de confiance et connue par votre navigateur.
{{< /alert >}}

![Proxmox ACME Staging Certificate](img/proxmox_certificates_staging_check_certificate.jpg)

#### 🏭 Certificat de production

Notre configuration est donc correcte, on peut générer notre certificat final. Pour cela, basculez sur votre compte ACME `ACME_PROD` afin d'être sur le directory `Let's Encrypt V2`. Puis commandez votre certificat via le bouton `Order Certificates Now` :

![Proxmox ACME Prod](img/proxmox_certificates_acme_prod.jpg)

Une nouvelle fois, si tout se passe bien, un enregistrement temporaire de type TXT qui commence par `_acme-challenge` devrait apparaitre dans votre zone DNS. Celui-ci a bien TTL de 5 minutes comme indiqué dans le paramètres `INFOMANIAK_TTL` de notre `ACME DNS Plugin` :

![Proxmox ACME Prod](img/infomaniak_acme_dns_record.jpg)

Le résultat de votre commande devrait être identique à celle sur le directory `Let's Encrypt V2 Staging` :

![Proxmox ACME Staging Success](img/proxmox_certificates_prod_success.jpg)

Et finalement, l'interface web Proxmox devrait se recharger avec votre nouveau certificat Let's Encrypt en place :

![Proxmox ACME Prod Certificate](img/proxmox_r3_certificate.jpg)

## 🎬 Conclusion

Nous avons pu voir ensemble les diverses étapes afin de signé l'interface graphique de notre hôte Proxmox.

Le renouvellement de votre certificat s'effectuera par défaut tout seul, 30 jours avant sa date d'expiration.
En cas de problème de renouvellement, vous revevrez un mail de Let's Encrypt sur le / les adresses mail que vous avez renseignées dans vos comptes ACME.

En cas de problème n'hésitez pas à jeter un coup d'oeil à la section [Problèmes et résolutions](#-problèmes-et-résolutions). Sinon, il y a plus qu'une chose à faire... [Apéééééro](https://estcequecestbientotlapero.fr/) !

![Pastis](img/pastis.webp)

## ❌ Problèmes et résolutions

### 💡 Infomaniak API token

Vous avez suivi la documentation du [site de go-acme.github.io](https://go-acme.github.io/lego/dns/infomaniak/index.html) pour votre domaine chez Infomaniak mais vous obtenez ce message d'erreur :
```shell
Please provide a valid Infomaniak API token in variable INFOMANIAK_API_TOKEN
 ```
![Proxmox Certificate Staging Failed](img/proxmox_certificates_staging_failed.jpg)

Et la configuration de votre `ACME DNS Plugin` resemble à cela :

![ACME Infomaniak wroonf config](img/acme_dns_plugin_wrong_config.jpg)

Alors la réponse est dans le message d'erreur. En fait le message nous indique clairement qu'il attend une variable avec la valeur `INFOMANIAK_API_TOKEN`. Pour une raison que j'ignore, sur Proxmox ce paramètre doit être nommé `INFOMANIAK_API_TOKEN` au lieu de `INFOMANIAK_ACESS_TOKEN`.

Au final, rien de bien méchant. Retournez sur la configuration de votre `ACME DNS Plugin` puis changez le nom de la valeur `INFOMANIAK_ACESS_TOKEN` en `INFOMANIAK_API_TOKEN` :

![ACME Infomaniak good config](img/acme_dns_plugin_good_config.jpg)

### 💡 Mauvais certificat

Il se peut que lors de la [génération du certificat de production](#-certificat-de-production), votre navigateur continu à vous afficher une alerte concernant votre certificat malgré que ce soit bien celui de Let's Encrypt sur le directory `Let's Encrypt V2` : 

![Proxmox alter good certificate](img/alert_good_certificate.jpg)

Bon dans un premier temps, vérifiez que votre certificat de production a bien été pris en compte.

Si c'est le cas, il y a de grandes chances pour que ce soit le cache de votre navigateur qui vous pose problème. Je vous invite à relancer votre navigateur et si cela ne suffit pas, vous pouvez effacer le cache de celui-ci et le relancer à nouveau.

Cela devrait résoudre votre problème 😉 !

### 💡 Validation ACME

Je n'ai pas encore eu  le cas avec Proxmox mais je pense que cela peut arriver.

 La configuration de votre `ACME DNS Plugin` est correcte puisque vous n'avez pas d'erreur particulière lors de la commande de votre certificat mais pourtant, la validation ACME échoue car elle a dépassé le temps imparti pour la validation (Timeout).

 Quelques pistes pour vous aider :
 - Venir jouer avec les valeurs des paramètres `Validation Delay` et / ou `INFOMANIAK_PROPAGATION_TIMEOUT` de votre `ACME DNS Plugin` :
 ![ACME DNS Plugin 60](img/acme_dns_plugin_60.jpg)
 - Changez provisoirement les DNS de votre hôte :
 ![Proxmox temp. DNS config.](img/proxmox_dns_config.jpg)