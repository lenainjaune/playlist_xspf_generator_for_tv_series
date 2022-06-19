# playlist_xspf_generator_for_tv_series
Générateur de playlist de série TV depuis une source (DVD pour le moment).

Objectif : 
j'ai un grand projet de faire de la place sur mes disques et garder les meilleurs versions de vidéos. 
La plupart de mes films en bonne qualité sont sur des DVDs du commerce que j'ai acquis. 
J'ai cloné ces DVDs dans des images ISO pour ne plus avoir à utiliser de lecteur DVD mécanique
 et parer aux dégradations éventuelles des supports qui restent fragiles. 
VLC et d'autres lecteurs permettent une bonne ergonomie pour la lecture de vidéos même depuis une image ISO. 
Poutant pour passer d'un DVD à l'autre, ce n'est pas pratique : il faut cliquer sur le DVD voulu, cliquer sur l'épisode, etc. 
L'objectif est de générer des playlists de tous les épisodes des images ISO et ainsi donner facilement accès à toute la série.

# Essai avec mes DVDs de Friends
Brouillon à réorganiser

J'ai acheté presque tous les DVDs de Friends (multi-langues) et ma compagne avait ceux qui me manquaient ; par ailleurs j'ai téléchargé presque tous les épisodes au format conteneur AVI. Au final toutes les vidéos AVI sont déjà sur les DVDs

	# Ce script permet donc de générer une playlist au format ouvert XSPF de tous les épisodes de Friends (en excluant les NON épisodes comme les bonus) depuis les fichiers images ISO des DVDs physiques.
	# Les épisodes seront identifiés selon la saison et le numéro dans la saison (si on a Internet on peut même obtenir les titres réels depuis Wikipédia).
	# Les commandes utiles pour lire avec VLC sont proposées en bas ainsi que la possibilité de configurer directement dans la playlist (par exemple pour avoir une piste en version FR sous-titrée EN - FRSTEN)


	# Nécessite lsdvd wget html2text


	# TODO : en faire un vrai script avec en param : DOSSIER="/media/VBIG/FILMS_ISO/" ; ID_SERIE="FRIENDS" 


	# TODO : vérifier l'universalité des playlists quand on utilise des extensions (voir en bas Configurer dans la playlist)

	# TODO : trouver mécanisme pour fusionner des épisodes comme s10x17 et x18

	# TODO : trancher sur l'utilité des noms des fichiers DVD BONUS exclus car il contiennent BONUS, puisqu'ils ne contiennent pas de pistes...

	# TODO :
	# _06@25,29 : titre 06 UNIQUEMENT pour 25 et 29
	# _06@25..29 : titre 06 UNIQUEMENT pour 25, 26, ... 29 (pas de tiret car trop utilisé)

	# TODO : faire que le séparateur de bloc ne soit plus le caractère souligné pour autoriser ttt à avoir ce caractère

	# Avant propos : 
	# - on distinguera ici le titre de la piste :
	#   - un titre (title) identifiera une vidéo sur un DVD (dans notre cas un DVD contient jusqu'à 7 titres comme l'indique la commande lsdvd) ; cette terminologie est utilisée par exemple par VLC quand on veut changer d'épisode
	#   - une piste (track) identifera une vidéo dans une playlist ; si un ensemble de pistes constitue une saison, on parlera plutôt en terme d'"épisode" (dans notre cas une saison contient jusqu'à 25 épisodes) ; cette terminologie est utilisée par exemple par le format de playlist XSPF
	# - le terme DVD reflète ici non pas le DVD physique, mais le fichier image DVD créé à partir d'un DVD physique (voir dessous exigences de nommage)

	# Les DVDs étant de sources différentes, épisodes diffèrent des titres des DVD (voir la correspondance dans le tableau dans le script).
	#  de plus dans la S10, il n'y a qu'un long épisode et non deux et un DVD est dédié aux bonus
	#  enfin j'ai constaté plus tard que s3 et s6 comportent 25 episodes_dvd au lieu de 24
	# => bref trop d'exceptions pour ne pas repenser le système

	# Exigences de nommage
	#
	# Le présent script fonctionnera UNIQUEMENT si les DVDs sont nommés dans une des formes suivantes :
	# - FRIENDS_S03_10_12.ISO (DVD de la saison 3 de FRIENDS des épisodes 10, 11, 12)
	# - FRIENDS_S10_BONUS.ISO (DVD sans épisodes)
	#
	# Astuce : si on ne souhaite pas toucher au nommage, on peut créer des liens CORRECTEMENT nommés et les utiliser à la place

	# Pour chaque saison de DVDs, refléter la table des règles d'inclusion/exclusion des titres (voir règles dessous)
	# Chaque ligne de la table constitue les règles pour une saison puisque les DVDs sont généralement vendus en coffret par saison et sont généralement organisés de la même manière.

	# Règles :
	#
	# - Chaque ligne identifiera de 1 à n blocs, s'il y en a plusieurs il seront séparés par un caractère souligné `_' (ex : 01_02_03_06_07@25 )
	# - Chaque ligne est une chaine qui commence et termine par un caractère souligné `_'  (ex : _01_02_03_)
	# - Chaque bloc est écrit selon une des formes suivantes :
	#    TTT[@[!]PPP[(.. ,)...]
	#    [!]ttt[,...]
	#
	#   TTT (composé de 1 à n chiffres) identifie un titre tel qu'il sera identifé par la commande lsdvd (voir la différence entre titre et piste dessus)
	#   PPP (composé de 1 à n chiffres) identifie une piste tel qu'elle sera identifiée dans le DVD (voir la différence entre titre et piste dessus)
	#   ttt (composé de 1 à n caractères textuels sauf le caractère souligné `_') identifie un texte pour permettre l'exclusion de certains DVDs (voir explications plus bas)
	#   ce qui se trouve entre crochets [ ] étant optionnel
	#   ce qui se trouve entre parenthèses ( ) étant une liste de choix (séparé par une espace ` ')
	#   ... indique que le sous-bloc est réitérable de 0 à n fois ; les itérations étant séparées par le caractère qui précède (soit point-point `..' soit virgule `,')
	#   
	#   Ainsi :
	#   - si le bloc est constitué uniquement par TTT, le titre numéro TTT sera pris en compte
	#   - si le bloc commence par TTT et est suivi par arobase `@', inclus ou non les DVDs qui contiennent ce titre, selon le mécanisme suivant :
	#     - si il y a PPP uniquement, le titre numéro TTT sera pris en compte UNIQUEMENT pour le DVD de la piste PPP
	#     - si il y a point d'exclamation `!' suivi de PPP, le titre numéro TTT sera pris en compte pour tous les DVDs SAUF celui de la piste PPP (l'inverse du précédent)
	#   - si le bloc commence par point d'exclamation `!' suivi de ttt, tous les DVDs qui contiennent ttt seront exclus (les DVDs qui ne contiennent pas d'épisodes : BONUS, MAKINGOFF, etc.)

	# Dans le cas de Friends, lsdvd indique 21 minutes environ pour un épisode


	# Obtenir les titres réels (nécessite Internet puisqu'on récupère une page de Wikipédia)
	# Nota : pour une raison ignorée html2text séquence par défaut les lignes par 79 caractères ; ajouter -width permet de contourner
	  
	DOSSIER="/media/VBIG/FILMS_ISO/" ; ID_SERIE="FRIENDS" ; \
	( \
	 IFS=$'\n' ; \
	 for l in $( wget --no-check-certificate -qO- \
	   https://fr.wikipedia.org/wiki/Liste_des_%C3%A9pisodes_de_Friends \
	   | grep -oE "<li><i>.+</i>)</li>" | sed 's/<[^>]*>//g' \
	   | sed -e 's/\&nbsp;/ /g'
	 ) ; do \
	  echo "$l" | html2text -utf8 -width ${#l} ; \
	 done ; \
	 unset IFS ; \
	) > "$DOSSIER/${ID_SERIE}_ALL_episodes.txt"


	# Générer la playlist

	DOSSIER="/media/VBIG/FILMS_ISO/" ; \
	ID_SERIE="FRIENDS" ; \
	TITRE_FR_ONLY="1" ; \
	DEBUG="0" ; \
	( \
	 beg="" ; \
	 beg="${beg}<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" ; \
	 beg="${beg} <playlist xmlns=\"http://xspf.org/ns/0/\" " ; \
	 beg="${beg}xmlns:vlc=" ; \
	 beg="${beg}\"http://www.videolan.org/vlc/playlist/ns/0/\" " ; \
	 beg="${beg}version=\"1\">\n" ; \
	 beg="${beg}  <title>Liste de tous les épisodes Friends</title>\n" ; \
	 beg="${beg}  <trackList>" ; \
	 echo -e "$beg" ; \
	 declare -A episodes_dvd ; \
	 echo "Règles (voir fonctionnement dessus)" &> /dev/null ; \
	 episodes_dvd[S01]='_01_02_03_' ; \
	 episodes_dvd[S02]='_01_02_03_' ; \
	 episodes_dvd[S03]='_01_02_03_06@25_' ; \
	 episodes_dvd[S04]='_01_02_03_' ; \
	 episodes_dvd[S05]='_01_02_03_' ; \
	 episodes_dvd[S06]='_01_02_03_06_07@25_' ; \
	 episodes_dvd[S07]='_01_02_03_06_' ; \
	 episodes_dvd[S08]='_03_04_06_07_' ; \
	 episodes_dvd[S09]='_02_03_04_05_' ; \
	 episodes_dvd[S10]='_03_04@!18_06@!18_07@!18_!BONUS_' ; \
	 episodes=$( \
	  find "$DOSSIER" -type f -iname "*$ID_SERIE*_episodes*.txt" \
	 ) ; \
	 ancienne_saison="00" ; \
	 episode_serie=1 ; \
	 IFS=$'\n' ; \
	 for dvd in $( \
	   find "$DOSSIER" -type f -iname "*$ID_SERIE*.ISO" | sort -u \
	  ) ; do \
	  nom_dvd=$( echo "$dvd" | grep -oE "[^/]+$" ) ; \
	  saison=$( echo "$nom_dvd" | cut -d '_' -f 2 ) ; \
	  if [[ "$ancienne_saison" != "$saison" ]] ; then \
		 episode_saison=1 ; \
		 ancienne_saison=$saison ; \
	  fi ; \
	  for titre in $( \
		lsdvd "$dvd" | grep ^Title | cut -d ' ' -f 2 | cut -d ',' -f 1 \
	  ) ; do \
	  [[ "$DEBUG" > "0" ]] && echo \
	   "$saison [[ _$titre =~ ${episodes_dvd[$saison]} ]] => $( \
		[[ ${episodes_dvd[$saison]} =~ _$titre ]] && echo oui || echo non \
	   )" ; \
	   if [[ ${episodes_dvd[$saison]} =~ _$titre ]] ; then \
		exceptions_piste=$( \
		 echo ${episodes_dvd[$saison]} | grep -oP "_$titre@\K([^_]+)" \
		) ; \
		exception_piste=$( echo $exceptions_piste | grep -oE "[0-9]+" ) ; \
		exceptions_DVD=$( \
		 echo ${episodes_dvd[$saison]} | grep -oP "_\!\K([^_]+)" \
		) ; \
		if [[ ! -z "$exceptions_DVD" ]] ; then \
		 if [[ $( echo "$dvd" | grep "$exceptions_DVD" ) ]] ; then \
		  [[ "$DEBUG" > "0" ]] && echo \
		   "DVD $dvd est exclus car il contient $exceptions_DVD" ; \
		  continue ; \    
		 fi ;
		fi ; \
		if [[ ! -z "$exceptions_piste" ]] ; then \
		 [[ "$DEBUG" > "0" ]] && echo \
		  "exceptions_piste : $exceptions_piste" ; \
		 if [[ "${exceptions_piste:0:1}" == "!" ]] ; then \
		  msg="" ; \
		  msg="${msg}titre _$titre à inclure SAUF pour la piste " ; \
		  msg="${msg}$exception_piste du DVD $nom_dvd" ; \
		  if [[ $( echo "$dvd" | grep "_$exception_piste" ) ]] ; then \
		   [[ "$DEBUG" > "0" ]] && echo "$msg => on n'affiche PAS" ; \
		   continue ; \
		  fi ; \
		  [[ "$DEBUG" > "0" ]] && echo "$msg => on affiche" ; \
		 else \
		  msg="" ; \
		  msg="${msg}titre _$titre à inclure UNIQUEMENT pour la piste " ; \
		  msg="${msg}$exception_piste du DVD $nom_dvd" ; \
		  if [[ ! $( echo "$dvd" | grep "_$exception_piste" ) ]] ; then \
		   [[ "$DEBUG" > "0" ]] && echo "$msg => on n'affiche PAS" ; \
		   continue ; \
		  fi ; \
		  [[ "$DEBUG" > "0" ]] && echo "$msg => on affiche" ; \
		 fi ; \
		fi ; \
		if [[ -f "$episodes" ]] ; then \
		 episode=$( cat "$episodes" | sed -n "${episode_serie}p" ) ; \
		 if [[ "$TITRE_FR_ONLY" > "0" ]] ; then \
		  episode=$( \
		   echo "$episode" \
		   | rev | grep -oP "^\)[^(]+\([[:space:]]*\K.+" | rev \
		 ) ; \
		 fi ; \
		fi ; \
		trk="" ; \
		trk="${trk}   <track>\n" ; \
		trk="${trk}    <location>" ; \
		trk="${trk}$( echo dvdsimple://$dvd#$titre )</location>\n" ; \
		trk="${trk}    <title>FRIENDS - ${saison}x" ; \
		trk="${trk}$( printf "%02d" $episode_saison ) - $episode" ; \
		trk="${trk}</title>\n" ; \
		trk="${trk}   </track>" ; \
		echo -e "$trk" ; \
		episode_saison=$(( episode_saison + 1 )) ; \
		episode_serie=$(( episode_serie + 1 )) ; \
	   fi ; \
	  done ; \
	 done ; \
	 unset IFS ; \
	 unset episodes_dvd ; \
	 total_episodes=$(( episode_serie - 1 )) ; \
	 end="" ; \
	 end="${end}   <"'!'"-- Total épisodes référencés ici : " ; \
	 end="${end}$total_episodes -->\n" ; \
	 end="${end}  </trackList>\n" ; \
	 end="${end} </playlist>" ; \
	 echo -e "$end" ; \
	) > "$DOSSIER/${ID_SERIE}_ALL.xspf"

	# Configurer VLC

	# VFSTEN
	vlc --sub-track=0 --audio-language=fr "$DOSSIER/${ID_SERIE}_ALL.xspf"

	# VOSTFR
	vlc --sub-track=1 --audio-language=en "$DOSSIER/${ID_SERIE}_ALL.xspf"

	Notes : 
	- Langage audio : fr ou en (sur certains DVD on peut utiliser d autes langues (lsdvd -x pour les lister)
	- Langage sous-titres : 0 (en) ou 1 (fr)


	# Configurer dans la playlist

	# Forcer la langue d'une piste audio (ici FR) et des sous-titres (ici EN) :
	# <track>
	#  <location>dvdsimple:///media/VBIG/FILMS_ISO/FRIENDS_S01_01_03.ISO#01</location>
	#  <title>FRIENDS - S01x01 - Celui qui déménage (The One Where Monica Gets A New Roommate)</title>
	#  <extension application="http://www.videolan.org/vlc/playlist/0">
	#   <vlc:option>audio-language=fr</vlc:option>
	#   <vlc:option>sub-track=0</vlc:option>
	#  </extension>
	# </track>
	# Nota : on peut aussi le faire pour la playlist complète (balise <playlist>)

	# Pour plus de possibilité de configurer dans la playlist pour VLC voir ici : https://wiki.videolan.org/XSPF/

