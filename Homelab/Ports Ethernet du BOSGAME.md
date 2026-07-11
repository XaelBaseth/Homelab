# Problème : ports Ethernet du BOSGAME non reconnus par OPNsense
## Matériel concerné : 
- BOSGAME E5 (Ryzen 5300U), 2 ports Ethernet 2.5G intégrés
## Logiciel : 
- OPNsense, FreeBSD 14.3-RELEASE-p10
## Symptôme
Après installation d'OPNsense, aucune interface Ethernet (re0/re1) n'apparaît dans ifconfig -a. Seule une interface Wi-Fi (rtw88, non désirée/non utilisée) était détectée, avec en prime un bug de driver qui spammait la console *rtw_tx failed to write TX skb to HCI -28*.
## Diagnostic
bashpciconf -lv | grep -B1 -A5 "RTL8125"
a révélé :
none2@pci0:2:0:0: ... vendor=0x10ec device=0x8125 ... rev=0x05
device = 'RTL8125 2.5GbE Controller'
Le préfixe none (au lieu de re) confirme que le contrôleur PCI est bien détecté au niveau matériel, mais aucun driver ne s'y attache.
## Cause racine
Le driver re (Realtek Ethernet) est compilé en dur dans le noyau FreeBSD de cette version d'OPNsense — ce n'est pas un module chargeable indépendant. Cette révision précise de la puce RTL8125 (rev=0x05) n'est pas reconnue par la table de compatibilité de ce driver intégré.
Tentative de correction avec le package alternatif realtek-re-kmod (driver Realtek officiel, packagé séparément) :
bashpkg add realtek-re-kmod-*.pkg
kldload if_re
→ Échec : module register: cannot register pci/re from if_re.ko; already loaded from kernel
Ce message confirme que le driver du noyau revendique déjà le nom re, empêchant le module alternatif de prendre le relais dynamiquement — un vrai conflit d'architecture, pas une erreur de manipulation.
## Recherche complémentaire
Confirmé par plusieurs retours de la communauté FreeBSD/OPNsense : cette combinaison précise (RTL8125 rev 0x05 + driver alternatif) est documentée comme problématique, parfois même quand le chargement du module alternatif réussit sur d'autres machines (rapports de "aucun trafic ne passe" ou d'instabilité nécessitant un redémarrage réseau manuel).
