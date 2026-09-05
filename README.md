# b64

[![Pylint CI](https://github.com/valorisa/b64/actions/workflows/pylint.yml/badge.svg)](https://github.com/valorisa/b64/actions/workflows/pylint.yml)
[![MarkdownLint CI](https://github.com/valorisa/b64/actions/workflows/markdownlint.yml/badge.svg)](https://github.com/valorisa/b64/actions/workflows/markdownlint.yml)

`b64` est un petit outil en ligne de commande pour **encoder** et **décoder** du texte en base64, en
respectant l'encodage **UTF-8**. Il est écrit en Python, n'utilise **que la bibliothèque standard**
(aucune installation de paquet supplémentaire), et tient dans un seul fichier.

Il sert à transformer un texte (avec accents, emoji, tirets cadratins, symboles mathématiques…) en
une chaîne base64 « propre » — c'est-à-dire une chaîne sur **une seule ligne**, composée uniquement
de lettres, de chiffres et de quelques signes (`+`, `/`, `=`) — puis à faire l'opération inverse.

## Pourquoi cet outil existe

Le base64 est un encodage qui représente n'importe quelle suite d'octets avec un alphabet restreint
et « sans surprise ». Comme il ne contient ni espace, ni retour à la ligne, ni caractère accentué,
il se **transporte** et se **colle** sans se déformer : c'est précieux quand on doit faire passer du
texte d'un endroit à un autre (un terminal, un chat, un fichier de configuration) sans risquer de
perdre des retours à la ligne ou de voir un caractère spécial mal interprété.

Le piège classique, c'est l'encodage des caractères. Un `é`, un `🅰️` ou un `—` ne tient pas sur un
seul octet : en UTF-8, ils occupent plusieurs octets. Un encodeur base64 qui ne sait pas qu'il
travaille en UTF-8 peut produire un résultat illisible au décodage (le fameux « mojibake »). `b64`
lit et écrit **toujours** en UTF-8, ce qui évite ce piège.

Enfin, `b64` applique une règle de prudence : le **résultat** (le base64, ou le texte décodé) est
écrit sur la sortie standard (`stdout`), pur, sans rien autour, pour qu'on puisse le copier ou le
rediriger tel quel. Les **informations de contrôle** (nombre de lignes, nombre d'octets) sont écrites
séparément, sur la sortie d'erreur (`stderr`). C'est un **filet de sécurité** : on voit tout de suite
si le compte de lignes correspond à ce qu'on attend, sans polluer la valeur qu'on copie.

## Ce qu'il fait

- Encoder du texte en base64 (depuis la ligne de commande, un fichier, ou un tube/pipe).
- Décoder du base64 en texte UTF-8 (même si le base64 a été remis en forme sur plusieurs lignes).
- Fonctionner en mode **interactif** (un menu) quand on le lance sans argument.
- Afficher un filet de vérification (compte de lignes et d'octets) sur `stderr`.

## Ce qu'il ne fait pas (volontairement)

- Il ne touche **pas au réseau** : tout se passe en local, sur ta machine.
- Il ne devine **pas** l'encodage : c'est UTF-8, point. (C'est un choix, pas un oubli.)
- Il ne remplace **pas** un éditeur de texte : c'est un convertisseur, pas un outil d'édition.

## Installation

Prérequis : Python 3 (présent sur Termux après `pkg install python`, et sur la plupart des
systèmes). Aucune dépendance externe : la bibliothèque standard suffit.

1. Placer le fichier `b64` quelque part, par exemple dans `~/bin/`.
2. Le rendre exécutable : `chmod +x ~/bin/b64`.
3. S'assurer que le dossier est dans le `PATH` (sinon, l'ajouter dans `~/.bashrc`) :

~~~bash
grep -q 'HOME/bin' ~/.bashrc || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/bin:$PATH"
which b64
~~~

La commande `which b64` doit afficher le chemin du fichier (par exemple `/data/.../home/bin/b64`).

## Utilisation

### Mode interactif

Lancé sans argument, `b64` ouvre un menu. On choisit `e` pour encoder, `d` pour décoder, `q` pour
quitter. Après `e` ou `d`, on saisit le texte sur autant de lignes qu'on veut ; on termine la saisie
par une ligne contenant **un seul point** (`.`), ou par la combinaison **Ctrl-D**.

~~~text
b64 — base64 UTF-8   [e]ncoder  [d]écoder  [q]uitter
> e
Texte à encoder :  (fin = ligne '.' seule, ou Ctrl-D)
~~~

Le résultat apparaît encadré par deux lignes `-----`, pour bien le délimiter à l'œil.

### Mode direct : encoder

Pour un texte court, on le passe avec `-t` (entre guillemets, pour le protéger du shell) :

~~~bash
b64 enc -t "bonjour le monde"
~~~

### Mode direct : décoder

On décode une chaîne base64 avec `-t` :

~~~bash
b64 dec -t "Ym9uam91ciBsZSBtb25kZQ=="
~~~

> Note : la chaîne ci-dessus est donnée **à titre d'illustration du format**. Pour un résultat
> garanti, décode toujours une valeur que **toi** tu as obtenue avec `b64 enc` (voir le roundtrip).

### Avec des fichiers

C'est le cas d'usage le plus utile. On encode un fichier entier :

~~~bash
b64 enc -f entree.txt > sortie.b64
~~~

Et on décode un fichier base64 (même s'il contient des retours à la ligne internes) :

~~~bash
b64 dec -f sortie.b64 > retrouve.txt
~~~

### Avec des tubes (pipes)

`b64` lit l'entrée standard si on ne donne ni `-t` ni `-f` :

~~~bash
cat entree.txt | b64 enc
b64 enc < entree.txt
~~~

## Le filet intégré (stderr)

À chaque appel, `b64` écrit une ligne de contrôle sur `stderr`, par exemple :

~~~text
# enc: 21 lines(wc) / 1234 bytes (utf-8) -> 1648 b64 chars
~~~

Les nombres **dépendent de ton texte** ; l'exemple montre seulement le **format**. L'intérêt : quand
tu encodes un fichier qui doit faire 21 lignes, tu vérifies d'un coup d'œil que le filet affiche bien
« 21 lines ». Si le compte ne colle pas, c'est que le contenu n'est pas celui que tu crois — et tu le
vois **avant** d'aller plus loin. C'est exactement l'esprit « mesurer, ne pas supposer ».

## Exemples concrets

### Roundtrip UTF-8 (la preuve que rien n'est perdu)

On encode un texte contenant des caractères multi-octets, on copie le base64 affiché, puis on le
décode : on doit retrouver **exactement** le texte de départ.

~~~bash
b64 enc -t "é 🅰️ — x ↦ M·x mod 256"
# copie la ligne base64 affichée, notons-la B, puis :
b64 dec -t "B"
~~~

Le résultat doit être identique au texte d'origine, accents, emoji et symboles compris.

### Roundtrip fichier → fichier (bit à bit)

Pour prouver que l'outil ne modifie rien, on encode puis on décode vers un second fichier, et on
compare les deux fichiers octet par octet avec `cmp` :

~~~bash
b64 enc -f original.txt > original.b64
b64 dec -f original.b64 > copie.txt
cmp original.txt copie.txt && echo IDENTIQUE
~~~

Si `cmp` ne dit rien et que « IDENTIQUE » s'affiche, les deux fichiers sont strictement identiques.

### Vérifier le nombre de lignes en encodant

Quand on veut s'assurer qu'un fichier a bien le bon nombre de lignes **et** en sortir le base64 en
une seule opération, le filet suffit :

~~~bash
b64 enc -f donnees.txt > donnees.b64
# la ligne stderr doit afficher le nombre de lignes attendu
~~~

## Limites honnêtes

- Le base64 d'un **très long** texte tient sur une longue ligne. Le copier-coller d'une ligne très
  longue dans certains terminaux peut la **tronquer** ; ce n'est pas un défaut de `b64`, c'est une
  limite du terminal. Dans ce cas, on préfère passer par des **fichiers** (`-f`) plutôt que par `-t`.
- `b64` travaille en UTF-8 uniquement. Décoder un base64 qui ne correspond pas à de l'UTF-8 valide
  provoque une **erreur explicite** (plutôt qu'un texte silencieux et faux) : c'est voulu.
- Le mode interactif lit depuis le clavier ; pour des contenus longs, le mode fichier est plus sûr.

## Licence

Distribué tel quel, sans garantie. Utilise-le, modifie-le, partage-le : c'est un petit outil pensé
pour être lu et compris, pas une boîte noire.
