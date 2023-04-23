Amendment Record :
#
#       16/11/2021 created by LDB, unification de GenConfPlt.sh et de son fork obsolete GenPltSshKeysCommon.sh
#
# Description :
#       Ce script permet de generer les clef ssh et les fichiers /etc/hosts pour une plate-forme donnee.
#       Ces informations sont generees dans le repertoire specifie et pour la plate-forme specifiee en parametres.
#
# PO :
# - lynxpo : pas de trust
# - lynx : trust des SRV vers les PO
# - ansible : trust uniquement le compte "sysansible@PosteSAM"
# SRV :
# - lynx : trust des SRV vers les SRV + trust lynxpo sur lui meme (localhost)
# - lynxweb : Pas de trust a l install & pas de clef
# - lynxpo : trust des lynxpo des PO & trust de lynxpologin des MONO
# - ansible : trust uniquement le compte "sysansible@PosteSAM"
# MONO :
# - lynx : trust des SRV vers les SRV + lynxpo sur lui meme (localhost)
# - lynxweb : Pas de trust a l install & pas de clef
# - lynxpo : trust de lynxpologin sur lui meme & trust de lynxpo des PO
# - lynxpologin : Pas de trust
# NONE :
# - lynxpo : pas de trust
# - lynx : trust des SRV vers les PO
# - ansible : trust uniquement le compte "sysansible@PosteSAM"
#
# Generation des clefs SSH pour une plateforme

UserList="lynx lynxpo lynxpologin ansible sysansible"
# ajout pour distribuer les cles ssh des lynxpologin MONO en post traitement
MonoList=" "

##########
# Suppression du domain name a la fin d'un hostname.
# $1 Hostname
# $2 Domain name
StripDomainFromHostname_ ()
{
Host=$1
Domain=$2
if [ "x$Domain" != "x" ]; then
        echo "${Host/%.$Domain/}"
else
        echo "$Host"
fi
}

##########
# Ajout du domain name a la fin d'un hostname, si besoin.
# $1 Hostname
# $2 Domain name
AddDomainToHostname_ ()
{
Host=$1
Domain=$2
if [ "x$Domain" != "x" ]; then
        Host="${Host/%.$Domain/}"
        echo "${Host}.$Domain"
else
        echo "$Host"
fi
}

# Obtention de la liste des postes SAM, prefixes PLT compris
# $1 Dir_rac
# $2 Plate-forme actuelle
GetPosteSam ()
{
        Dir_rac="$1"
        PLT="$2"

        POSTE_SAM=$(grep -w "${PLT}SAM" "${Dir_rac}/conf/PlatformDesc.conf" | cut -d' ' -f4)
        POSTE_SAM2=
        IFS=","
        for poste in ${POSTE_SAM}; do
                # 3 cas: "poste" sans PLT, "$PLT/poste" (plate-forme actuelle), "PLT2/poste" (autre plate-forme).
                PltSam=$(echo "$poste" | cut -d'/' -f1)
                if [ "x$POSTE_SAM" != "x$PltSam" ]; then
                        # Cas 2 ou 3: il y a une PLT dans $poste.
                        # poste est une entree de la forme <PLT actuelle>/$Mch_Name.
                        fullposte="$poste"
                        # Couper cette partie "PLT/" en debut de cette entree de POSTE_SAM.
                        poste2=$(echo "$poste" | cut -d'/' -f2)
#                       if [ "x${PLT}/$poste2" = "x$poste" ]; then
#                               # Cas 2: plate-forme actuelle
#                       else
#                               # Cas 3: autre plate-forme.
#                       fi
                else
                        # Cas 1: pas de PLT dans $poste
                        PltSam="$PLT"
                        fullposte="${PLT}/$poste"
                        poste2="$poste"
                fi

                # Version de POSTE_SAM avec des entrees indiquant la plate-forme, et des espaces, pour faciliter l'enumeration (pas besoin de jouer avec IFS).
                if [ "x$POSTE_SAM2" = "x" ]; then
                        POSTE_SAM2="$fullposte"
                else
                        POSTE_SAM2="$POSTE_SAM2 $fullposte"
                fi
                echo "$poste2" >> SAM_ansible_host_${PLT}
        done
        echo "$POSTE_SAM2"
}

################################################
# Internal function
# GenKeyUser
# Input  : User & GenAutho
# Output : Set up the associated SSH configuration file
#
GenKeyUser ()
{
        lpos=${PWD}
        use="$1"
        GenAutho="$2"
        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Generation clefs user $use pour la machine ${srv}"
        mkdir -p ${use}
        cd ${use}
        ssh-keygen -q -C ${use}@${srv} -f ${use}@${srv} -t rsa -b ${RSASizeKey} -P ""
        echo "IdentityFile ~/.ssh/${use}@${srv}" > config
        echo "SendEnv LDISPLAY LPONAME LFINGERPRINT LBASCUL" >> config
        if [ "$GenAutho" = "YES" ]; then
                echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Ajout de la public key user $use pour la machine ${srv}"
                KUpub=$(cat ${use}@${srv}.pub)
                echo "from=\"localhost,::1,127.0.0.1,${KhIp},${khName}\" ${KUpub}" >> ${PosO}/authorized_keys_${use}
        fi
        # Multiplexage SSH pour lynx sur tous les SRV & MONO.
        if [ "x$use" = "xlynx" ]; then
                if [ "x$TYPE_MCH_SYS" = "xSRV" ] || [ "x$TYPE_MCH_SYS" = "xMONO" ]; then
                        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Ajout multiplexage SSH user $use pour la machine ${srv}"
                        echo "" >> config
                        host_list=
                        for item in ${UPLT_SRV_NAMES["$PLT"]}; do
                                item_S=$(StripDomainFromHostname_ "$item" "$DOMAIN2DEF")
                                item_L=$(AddDomainToHostname_ "$item" "$DOMAIN2DEF")
                                host_list="$host_list $item_S"
                                if [ "x$item_S" != "x$item_L" ]; then
                                        host_list="$host_list $item_L"
                                fi
                        done
                        for item in ${UPLT_MONO_NAMES["$PLT"]}; do
                                item_S=$(StripDomainFromHostname_ "$item" "$DOMAIN2DEF")
                                item_L=$(AddDomainToHostname_ "$item" "$DOMAIN2DEF")
                                host_list="$host_list $item_S"
                                if [ "x$item_S" != "x$item_L" ]; then
                                        host_list="$host_list $item_L"
                                fi
                        done
                        echo "host $host_list" >> config
                        echo "    ControlMaster auto" >> config
                        echo "    Controlpath ~/.ssh/controlmasters/%r@%h:%p" >> config
                        echo "    ControlPersist 10m" >> config

                fi
        fi
        cd ${lpos}
}

# TODO Eclater cette fonction monolithique en au moins 3 morceaux:
# * le gros du code actuel;
# * une fonction pour generer la liste des hosts de toutes les plate-formes;
# * une fonction pour generer les ansible_hosts sur la base des listes des hosts de toutes les plate-formes.

GenPlatform ()
{
        Dir_rac="$1"
        GPLT="$2"

        if [ ! -d "$Dir_rac" -o "x$GPLT" = "x" ]
        then
                echo "Erreur interne"
                exit 1
        fi

        PosO=${PWD}
        declare -A UPLT_PO_NAMES
        declare -A UPLT_PO_IPADDRS
        declare -A UPLT_SRV_NAMES
        declare -A UPLT_SRV_IPADDRS
        declare -A UPLT_MONO_NAMES
        declare -A UPLT_MONO_IPADDRS
        declare -A UPLT_NONE_NAMES
        declare -A UPLT_NONE_IPADDRS
#       declare -A PLT_FOR_SRV

        ###############
        # Initialisation
        # On cree les rep de chaque plateforme
        # On initialise le fichier host
        # On commence a initialiser des listes de machines, en verifiant au passage leur type
        for uplt in $(echo ${GPLT} | sed 's/|/ /g')
        do
                rm -rf ./${uplt}
                mkdir ./${uplt}
#               cat << EOF > ./${uplt}/hosts_${uplt}
## Table host generee automatiquement par ${appli} le $(date '+%d-%m-%Y %H:%M')
##
#127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
#EOF

                # Pour simplifier le script, on strippe le domain s'il est present dans le premier hostname de la liste.
                DOMAIN2DEF="$(grep ${uplt}DOMAIN ${Dir_rac}/conf/PlatformDesc.conf | awk '{print $4}')"
                if [ "x$DOMAIN2DEF" != "x" ]; then
                        uplt_ALL=$(grep -Ew ${uplt} ${Dir_rac}/conf/PlatformDesc.conf | grep -v "^#" | awk -F';' '{print $2}' | awk -F',' '{print $1}' | sed -e "s/.$DOMAIN2DEF//")
                else
                        uplt_ALL=$(grep -Ew ${uplt} ${Dir_rac}/conf/PlatformDesc.conf | grep -v "^#" | awk -F';' '{print $2}' | awk -F',' '{print $1}')
                fi
                echo "$uplt_ALL"
                for srv in $uplt_ALL; do
                        TYPE_MCH_SYS=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $6}' | awk -F',' '{print $1}')
                        case ${TYPE_MCH_SYS} in
                        'PO')
                                val=${UPLT_PO_NAMES["$uplt"]}
                                # Nouvelle plate-forme / premier PO dans cette plate-forme ?
                                if [ "x$val" = "x" ]; then
                                        UPLT_PO_NAMES["$uplt"]="$srv"
                                else
                                        UPLT_PO_NAMES["$uplt"]="$val $srv"
                                fi
                        ;;
                        'SRV')
                                val=${UPLT_SRV_NAMES["$uplt"]}
                                # Nouvelle plate-forme / premier SRV dans cette plate-forme ?
                                if [ "x$val" = "x" ]; then
                                        UPLT_SRV_NAMES["$uplt"]="$srv"
                                else
                                        UPLT_SRV_NAMES["$uplt"]="$val $srv"
                                fi
                        ;;
                        'MONO')
                                val=${UPLT_MONO_NAMES["$uplt"]}
                                # Nouvelle plate-forme / premier MONO dans cette plate-forme ?
                                if [ "x$val" = "x" ]; then
                                        UPLT_MONO_NAMES["$uplt"]="$srv"
                                else
                                        UPLT_MONO_NAMES["$uplt"]="$val $srv"
                                fi
                        ;;
                        'NONE')
                                val=${UPLT_NONE_NAMES["$uplt"]}
                                # Nouvelle plate-forme / premier NONE dans cette plate-forme ?
                                if [ "x$val" = "x" ]; then
                                        UPLT_NONE_NAMES["$uplt"]="$srv"
                                else
                                        UPLT_NONE_NAMES["$uplt"]="$val $srv"
                                fi
                        ;;
                        *)
                        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Erreur serveur ${srv} a un type machine non connue $TYPE_MCH_SYS"
                        ;;
                esac
#               PLT_FOR_SRV["$srv"]="$uplt"

                done
        done

        # on initialise les fichiers authorized_keys_ & known_hosts
        for use in ${UserList}
        do
                >authorized_keys_${use}
        done
        >known_hosts

        echo "---> Generation pour la ou les plateforme(s) ${GPLT} ..."
        # Liste des hosts avec leur nom primaire, qui peut contenir un nom de domaine.
        # XXX uplt ici auparavant pour le fichier de destination, ca paraissait curieux...
        grep -Ew ${GPLT} ${Dir_rac}/conf/PlatformDesc.conf | grep -v "^#" | awk -F';' '{print $2}' | awk -F',' '{print $1}' > ./${GPLT}/list_hosts_${GPLT}.lst

        # Pour simplifier le script, on strippe le domain s'il est present dans le premier hostname de la liste.
        DOMAIN2DEF="$(grep ${GPLT}DOMAIN ${Dir_rac}/conf/PlatformDesc.conf | awk '{print $4}')"
        if [ "x$DOMAIN2DEF" != "x" ]; then
                LISTE_DES_MACHINES=$(grep -Ew ${GPLT} ${Dir_rac}/conf/PlatformDesc.conf | grep -v "^#" | awk -F';' '{print $2}' | awk -F',' '{print $1}' | sed -e "s/.$DOMAIN2DEF//")
        else
                LISTE_DES_MACHINES=$(grep -Ew ${GPLT} ${Dir_rac}/conf/PlatformDesc.conf | grep -v "^#" | awk -F';' '{print $2}' | awk -F',' '{print $1}')
        fi

        BadHostname=""
        for srv in ${LISTE_DES_MACHINES}; do
                [[ $srv =~ [[:punct:]] ]] && [[ ! "$srv" =~ [-] ]] && BadHostname="$BadHostname $srv"
        done

        if [ -n "${BadHostname}" ]; then
                echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Le(s) hostname ne respecte pas les caracteres autorises [a-z],[A-Z],[0-9] et (-)"
                echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- ${BadHostname}"
                zenity --error --text="Bad hostname. Generation des clefs abandonnee"
                exit 1
        fi

        for srv in ${LISTE_DES_MACHINES}; do
                #GetInfoSrv ${srv}
                PLT=$(grep \;${srv}[\;\,\.] "${Dir_rac}/conf/PlatformDesc.conf" | awk -F';' '{print $1}')
                # Nom primaire brut du host, sans stripper l'eventuel domaine.
                srvreal1=$(grep \;${srv}[\;\,\.] "${Dir_rac}/conf/PlatformDesc.conf" | grep -v "^#" | awk -F';' '{print $2}' | awk -F',' '{print $1}')
                POSTE_SAM=$(GetPosteSam "$Dir_rac" "$PLT")
                RSASizeKey="$(grep -w "${PLT}RSASK" "${Dir_rac}/conf/PlatformDesc.conf" | cut -d' ' -f4)"
                [ -z "${RSASizeKey}" ] && RSASizeKey=4096 # on met par defaut la taille a 4096

                cd ${PLT}
                rm -rf ${srv}
                mkdir ${srv}
                cd ${srv}

                echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Generation des clefs pour la machine ${srv}"
                ssh-keygen -q -C ${srv} -f ${srv} -t rsa -b ${RSASizeKey} -P ""
                Kpub=$(cat ${srv}.pub)

                # Ajout des informations sur la machine dans le fichier /etc/hosts
                NbIf=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $3}')
                blockIfs=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $4}')
                TYPE_MCH_SYS=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $6}' | awk -F',' '{print $1}')
                TYPE_MCH_LYNX=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $6}' | awk -F',' '{print $2}')
                if [[ $(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $2}'| awk -F',' '{print NF}') -ne 1 ]]
                then
                        ALIAS=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $2}'|cut -d',' -f2-)
                else
                        ALIAS=""
                fi
                [[ -n "${ALIAS}" ]] && echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Alias : ${ALIAS}"

                NumIfIp="0" # Numero d'interface avec une adresse Ip. On ne compte pas les DHCP

                for (( i = 1; i <= ${NbIf}; i+=1))
                do
                        # On recupere le block de l'interface nif=${i}
                        IfElem=$(echo ${blockIfs} | awk -F'/' -v nif=${i} '{print $(nif) }')

                        # Recuperation de l'adresse Ip sur le block
                        adrIp=$(echo ${IfElem} | cut -d, -f3)

                        # On recherche un eventuel nom de domaine associe
                        DOMAIN2DEF="$(grep ${PLT}DOMAIN ${Dir_rac}/conf/PlatformDesc.conf | awk '{print $4}')"

                        # On ne fait pas d'ajout dans /etc/hosts dans le cas d'une configuration en DHCP
                        if [ "${adrIp}" != "DHCP" ]
                        then
                                NumIfIp=$( expr ${NumIfIp} + 1 )
                                case ${NumIfIp} in
                                        '1') # Cas de l'ip principale (premiere ip significative lue) porte le nom de base de la machine
#                                               liste_srv="${srv}"
                                                KhIp="${adrIp}"
                                                khName="${srvreal1}"
                                                [ -n "${ALIAS}" ] && khName="${khName},${ALIAS}"
#                                               if [ -n "${DOMAIN2DEF}" ]; then
#                                                       liste_srv="${liste_srv} ${srv}.${DOMAIN2DEF}"
#                                               fi
                                                ;;

                                        '2') # Cas de l'ip secondaire (deuxieme ip significative lue) porte le nom de la machine suffixe par Alt
#                                               liste_srv="${srv}Alt"
                                                KhIp="${KhIp},${adrIp}"
# Ca suppose que $srv est court, ce qui nous arrange.
                                                khName="${khName},${srv}Alt"
                                                if [[ -n "${ALIAS}" ]]
                                                then
                                                        # Il faut generer les alias Alt
                                                        AliasAltW=""
                                                        IFS=',' # pour travailler sur la , comme separateur
                                                        for Walias in ${ALIAS}
                                                        do
                                                                # Si l'alias a un . c'est un alias avec nom de domaine.
                                                                # Donc il faut rajouter Alt avant le premier .
                                                                # Sinon on rajoute Alt a la fin de la string
                                                                if [[ -n "$(echo ${Walias} | grep '\.' )" ]]
                                                                then
                                                                        AliasAltW="${AliasAltW} $(echo ${Walias} | sed 's/\./Alt\./')"
                                                                else
                                                                        AliasAltW="${AliasAltW} $(echo ${Walias} | sed 's/$/Alt/')"
                                                                fi
                                                        done
                                                        unset IFS
                                                        AliasAlt=$(echo ${AliasAltW} | sed -e 's/^ //' -e 's/ /,/g')
                                                        khName="${khName},${AliasAlt}"
                                                fi
#                                               if [ -n "${DOMAIN2DEF}" ]; then
#                                                       liste_srv="${liste_srv} ${srv}Alt.${DOMAIN2DEF}"
#                                               fi
                                                ;;

                                        * ) # Cas au dela
#                                               liste_srv="${srv}-${NumIfIp}"
                                                KhIp="${KhIp},${adrIp}"
# Ca suppose que $srv est court, ce qui nous arrange.
                                                khName="${khName},${srv}-${NumIfIp}"
#                                               if [ -n "${DOMAIN2DEF}" ]; then
#                                                       liste_srv="${liste_srv} ${srv}-${NumIfIp}.${DOMAIN2DEF}"
#                                               fi
                                                ;;
                                esac
                                # Ecriture de la ligne construite dans le futur /etc/hosts
#                               echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Ecriture dans hosts_${PLT} pour ${liste_srv}"
#                               echo "${adrIp}  ${liste_srv}" >> ${PosO}/${PLT}/hosts_${PLT}
                        else # En DHCP il faut mettre le nom de la machine dans khName & 127.0.0.1 dans KhIp
                                khName=${srvreal1}
                                KhIp="127.0.0.1"
                        fi
                done

                # ALIAS=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $2}'|cut -d',' -f2-)
                # if [[ -n "${ALIAS}" ]]
                # then
                        # khName="${khName},${ALIAS}"
                # fi

                # creation du known_hosts
                echo "localhost,${KhIp},${khName} ${Kpub}"      >> ${PosO}/known_hosts
                echo "localhost,${KhIp},${khName}" >IP.tmp

                # creation des clef ssh pour les users et (YES/NO) gene des authorized_keys
                case ${TYPE_MCH_SYS} in
                        'PO')
                                if [ "x$NumIfIp" != "x0" ]; then
                                        val=${UPLT_PO_IPADDRS["$PLT"]}
                                        if [ "x$val" = "x" ]; then
                                                UPLT_PO_IPADDRS["$PLT"]="${KhIp}"
                                        else
                                                UPLT_PO_IPADDRS["$PLT"]="${val}, ${KhIp}"
                                        fi
                                fi
                                GenKeyUser lynxpo YES
                                mkdir -p lynx # On a besoin du home de lynx de facon a poser les authorized_keys. Mais on n a pas de clef.
                                mkdir -p ansible
                        ;;
                        'SRV')
                                if [ "x$NumIfIp" != "x0" ]; then
                                        val=${UPLT_SRV_IPADDRS["$PLT"]}
                                        if [ "x$val" = "x" ]; then
                                                UPLT_SRV_IPADDRS["$PLT"]="${KhIp}"
                                        else
                                                UPLT_SRV_IPADDRS["$PLT"]="${val}, ${KhIp}"
                                        fi
                                fi
                                GenKeyUser lynxpo NO
                                GenKeyUser lynx YES
                                mkdir -p ansible
                        ;;
                        'MONO')
                                if [ "x$NumIfIp" != "x0" ]; then
                                        val=${UPLT_MONO_IPADDRS["$PLT"]}
                                        if [ "x$val" = "x" ]; then
                                                UPLT_MONO_IPADDRS["$PLT"]="${KhIp}"
                                        else
                                                UPLT_MONO_IPADDRS["$PLT"]="${val}, ${KhIp}"
                                        fi
                                fi
                                GenKeyUser lynxpo NO
                                GenKeyUser lynx YES
                                GenKeyUser lynxpologin YES
                                mkdir -p ansible
                                MonoList=$MonoList" "$srv
                        ;;
                        'NONE')
                                if [ "x$NumIfIp" != "x0" ]; then
                                        val=${UPLT_NONE_IPADDRS["$PLT"]}
                                        if [ "x$val" = "x" ]; then
                                                UPLT_NONE_IPADDRS["$PLT"]="${KhIp}"
                                        else
                                                UPLT_NONE_IPADDRS["$PLT"]="${val}, ${KhIp}"
                                        fi
                                fi
                                GenKeyUser lynx YES
                                mkdir -p ansible
                        ;;
                        *)
                        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Erreur serveur ${srv} a un type machine non connue $TYPE_MCH_SYS"
                        ;;
                esac

                # Generation d'une paire de cles pour sysansible sur les SAM qui appartiennent a la plate-forme actuelle.
                for poste in ${POSTE_SAM}; do
                        if [ "x$poste" = "${PLT}/${srv}" ]; then
                                GenKeyUser sysansible yes
                        fi
                done
                cd ${PosO}
        done

        # copie du known_hosts et authorized_keys par serveur
        for srv in ${LISTE_DES_MACHINES}; do
                TYPE_MCH_SYS=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $6}' | awk -F',' '{print $1}')
                PLT=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $1}')
                POSTE_SAM=$(GetPosteSam "$Dir_rac" "$PLT")
                cd ${PLT}
                case ${TYPE_MCH_SYS} in
                        'PO')
                                for use in ${UserList}
                                do
                                        if [ -d ${srv}/${use} ]
                                        then
                                                if [[ -e ../authorized_keys_${use} ]]
                                                then
                                                        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- mise en place de authorized_keys ${use} sur ${TYPE_MCH_SYS} ${srv}"
                                                        cp ../authorized_keys_${use} ${srv}/${use}/authorized_keys
                                                fi
                                                cp ../known_hosts ${srv}/${use}
                                        fi
                                done
                                # Ajout dans la liste, sauf si c'est un SAM.
                                (echo "${POSTE_SAM}" | grep -w "${PLT}/$srv") || (echo "$srv" >> ../PO_ansible_host_${PLT})
                        ;;
                        'SRV')
                                for use in ${UserList}
                                do
                                        if [ -d ${srv}/${use} ]
                                        then
                                                if [[ -e ../authorized_keys_${use} ]]
                                                        then
                                                        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- mise en place de authorized_keys ${use} sur ${TYPE_MCH_SYS} ${srv}"
                                                        cp ../authorized_keys_${use} ${srv}/${use}/authorized_keys
                                                fi
                                                cp ../known_hosts ${srv}/${use}
                                        fi
                                done
                                # ajout de lynxpo dans authorized_keys de lynx
                                use="lynxpo"
                                echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Rajout key pub user $use serveur ${srv} sur lynx de ${srv}"
                                Kpub=$(cat ${srv}/${use}/${use}@${srv}.pub)
                                Kip=$(cat ${srv}/IP.tmp)
                                echo "from=\"127.0.0.1,::1,${Kip}\" ${Kpub}" >> ${srv}/lynx/authorized_keys

                                # creation d'un authorized_keys vide pour lynxweb
                                use="lynxweb"
                                mkdir -p ${srv}/${use}
                                touch ${srv}/${use}/authorized_keys
                                # Ajout dans la liste, sauf si c'est un SAM.
                                (echo "${POSTE_SAM}" | grep -w "${PLT}/$srv") || (echo "$srv" >> ../SRV_ansible_host_${PLT})
                        ;;
                        'MONO')
                                for use in ${UserList}
                                do
                                        if [ -d ${srv}/${use} ]
                                        then
                                                if [[ -e ../authorized_keys_${use} ]]
                                                then
                                                        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- mise en place de authorized_keys ${use} sur ${TYPE_MCH_SYS} ${srv}"
                                                        cp ../authorized_keys_${use} ${srv}/${use}/authorized_keys
                                                fi
                                                cp ../known_hosts ${srv}/${use}
                                        fi
                                done
                                # ajout de lynxpo dans authorized_keys de lynx
                                use="lynxpo"
                                echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Rajout key pub user $use serveur ${srv} sur lynx de ${srv}"
                                Kpub=$(cat ${srv}/${use}/${use}@${srv}.pub)
                                Kip=$(cat ${srv}/IP.tmp)
                                echo "from=\"127.0.0.1,::1,${Kip}\" ${Kpub}" >> ${srv}/lynx/authorized_keys

                                # ajout de lynxpologin dans authorized_keys de lynxpo
                                use="lynxpologin"
                                echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Rajout key pub user $use serveur ${srv} sur lynxpo de ${srv}"
                                Kpub=$(cat ${srv}/${use}/${use}@${srv}.pub)
                                Kip=$(cat ${srv}/IP.tmp)
                                echo "from=\"127.0.0.1,::1,${Kip}\" ${Kpub}" >> ${srv}/lynxpo/authorized_keys

                                # creation d'un authorized_keys vide pour lynxweb
                                use="lynxweb"
                                mkdir -p ${srv}/${use}
                                touch ${srv}/${use}/authorized_keys
                                # Ajout dans la liste, sauf si c'est un SAM.
                                (echo "${POSTE_SAM}" | grep -w "${PLT}/$srv") || (echo "$srv" >> ../MONO_ansible_host_${PLT})
                        ;;
                        'NONE')
                                for use in ${UserList}
                                do
                                        if [ -d ${srv}/${use} ]
                                        then
                                                if [[ -e ../authorized_keys_${use} ]]
                                                        then
                                                        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- mise en place de authorized_keys ${use} sur ${TYPE_MCH_SYS} ${srv}"
                                                        cp ../authorized_keys_${use} ${srv}/${use}/authorized_keys
                                                fi
                                                cp ../known_hosts ${srv}/${use}
                                        fi
                                done
                                # Ajout dans la liste, sauf si c'est un SAM.
                                (echo "${POSTE_SAM}" | grep -w "${PLT}/$srv") || (echo "$srv" >> ../NONE_ansible_host_${PLT})
                        ;;
                        *)
                        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Erreur serveur ${srv} a un type machine non connue $TYPE_MCH_SYS"
                        ;;
                esac
                # ajout de "sysansible" dans authorized_keys de "ansible"
                for poste in $POSTE_SAM; do
                        PltSam=$(echo "$poste" | cut -d'/' -f1)
                        poste2=$(echo "$poste" | cut -d'/' -f2)
                        use="sysansible"
                        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Rajout key pub du user ${use} de ${poste} sur ansible de ${srv}"
                        Kpub=$(cat ../${PltSam}/${poste2}/${use}/${use}@${poste2}.pub)
                        Kip=$(cat ../${PltSam}/${poste2}/IP.tmp)
                        echo "from=\"127.0.0.1,::1,${Kip}\" ${Kpub}" >> ${srv}/ansible/authorized_keys
                done
                cd ${PosO}
        done

        # modification du fichier authorized_keys par serveur pour ajout lynxpologin@MONO
        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Rajout des clefs lynxpologin du ou des MONO dans les SRV et autres MONO"
        for monosrv in ${MonoList}; do
                for srv in ${LISTE_DES_MACHINES}; do
                        TYPE_MCH_SYS=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $6}'| awk -F',' '{print $1}')
                        PLT=$(grep \;${srv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $1}')
                        MONOPLT=$(grep \;${monosrv}[\;\,\.] ${Dir_rac}/conf/PlatformDesc.conf | awk -F';' '{print $1}')
                        cd ${PLT}
                        if [[ "$TYPE_MCH_SYS" = "SRV" || "$TYPE_MCH_SYS" = "MONO" ]]; then
                                # ajout de lynxpologin dans authorized_keys de lynxpo - cas d une machine mono
                                use="lynxpologin"
                                if [ "$monosrv" != "$srv" ]; then
                                        echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Rajout key pub user $use serveur ${monosrv} sur lynxpo de ${srv}"
                                        Kpub=$(cat ../${MONOPLT}/${monosrv}/${use}/${use}@${monosrv}.pub)
                                        Kip=$(cat ../${MONOPLT}/${monosrv}/IP.tmp)
                                        echo "from=\"127.0.0.1,::1,${Kip}\" ${Kpub}" >> ${srv}/lynxpo/authorized_keys
                                fi
                        fi
                        cd ${PosO}
                done
        done

        rm known_hosts authorized_keys_*
        find . -name IP.tmp -exec rm -f {} \;

        # generation du fichier ansible_hosts
        # TODO c'est plus complique: l'ansible_hosts d'une plate-forme doit contenir des infos pour toutes les plate-formes gerees par ce SAM.
        for uplt in $(echo ${GPLT} | sed 's/|/ /g'); do
                echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Generation du fichier ansible_hosts"

                if [ -e ${PosO}/SAM_ansible_host_${uplt} ]; then
                        echo -e "\n[sam]" >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                        sort < ${PosO}/SAM_ansible_host_${uplt} | uniq >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                fi

                if [ -e ${PosO}/SRV_ansible_host_${uplt} ]; then
                        echo -e "\n[serveur]" >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                        cat ${PosO}/SRV_ansible_host_${uplt} >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                fi

                if [ -e ${PosO}/PO_ansible_host_${uplt} ]; then
                        echo -e "\n[po]" >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                        cat ${PosO}/PO_ansible_host_${uplt} >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                fi

                if [ -e ${PosO}/MONO_ansible_host_${uplt} ]; then
                        echo -e "\n[mono]" >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                        cat ${PosO}/MONO_ansible_host_${uplt} >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                fi

                if [ -e ${PosO}/NONE_ansible_host_${uplt} ]; then
                        echo -e "\n[none]" >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                        cat ${PosO}/NONE_ansible_host_${uplt} >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                fi

                echo " " >> ${PosO}/${uplt}/ansible_hosts_${uplt}
                rm -f ${PosO}/*_ansible_host_${uplt}
        done

        # Generation du fichier nftables.conf
        for uplt in $(echo ${GPLT} | sed 's/|/ /g'); do
                mkdir -p "${PosO}/${uplt}"
                NFTABLESCONF="${PosO}/${uplt}/nftables.conf"
                cp -Lp ${Dir_rac}/conf/nftables.conf "$NFTABLESCONF"
                echo "$(date '+%d-%m-%Y %H:%M:%S')-${appli}- Generation du fichier nftables.conf pour la plate-forme $uplt"
                val=${UPLT_SRV_IPADDRS["$uplt"]}
                if [ "x$val" != "x" ]; then
                        sed -i -e "s/SRVIPADDRLIST/${val}/g" "$NFTABLESCONF"
                else
                        echo "Impossible de trouver des adresses IP pour les serveurs de la plate-forme $uplt !"
                        sed -i -e 's/define SRV/#define SRV/g' "$NFTABLESCONF"
                        sed -i -e 's/tcp dport 11130 ip saddr $SRV /tcp dport 11130 /g' "$NFTABLESCONF"
                fi
                val=${UPLT_PO_IPADDRS["$uplt"]}
                if [ "x$val" != "x" ]; then
                        sed -i -e "s/POIPADDRLIST/${val}/g" "$NFTABLESCONF"
                else
                        echo "Impossible de trouver des adresses IP pour les postes operateur de la plate-forme $uplt !"
                        sed -i -e 's/define PO/#define PO/g' "$NFTABLESCONF"
                fi
                val=${UPLT_MONO_IPADDRS["$uplt"]}
                if [ "x$val" != "x" ]; then
                        sed -i -e "s/MONOIPADDRLIST/${val}/g" "$NFTABLESCONF"
                else
                        echo "Impossible de trouver des adresses IP pour les MONOs de la plate-forme $uplt !"
                        sed -i -e 's/define MONO/#define MONO/g' "$NFTABLESCONF"
                fi
        done
}
