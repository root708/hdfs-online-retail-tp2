# TP2 HDFS : Online Retail II sur un cluster Hadoop local

TP du module **Big Data & Hadoop** (École d'ingénieurs ISM, L3-IA).
Objectif : démarrer un HDFS local, y déposer un fichier volumineux, observer son découpage en blocs et tester la tolérance aux pannes.

**Auteur :** Papa Arona SENE, L3-IA

## Environnement

- Windows 11 avec Ubuntu sous WSL (Python 3.12)
- Hadoop installé dans `/usr/local/hadoop` (mode pseudo-distribué : 1 NameNode + 1 DataNode)
- Jeu de données : [Online Retail II (UCI)](https://archive.ics.uci.edu/dataset/502/online+retail+ii), fichier `online_retail_II.xlsx` (44 Mo), non versionné dans ce dépôt

## Contenu du dépôt

```
scripts/                         commandes du TP, dans l'ordre
  00_reset_hdfs.sh               remise à zéro (dépannage)
  01_start_hdfs.sh               démarrage et vérification
  02_prepare_dataset.sh          zip -> xlsx -> csv -> gros csv
  03_load_hdfs.sh                dépôt et lecture dans HDFS
  04_inspect_and_fault_test.sh   fsck et test de panne
docs/screenshots/                19 captures numérotées dans l'ordre chronologique
docs/TP2_HDFS_Papa_Arona_SENE.pdf   livrable rendu
```

## Déroulé

### 1. Démarrage de HDFS et dépannage du DataNode

| Étape | Capture | Ce qu'on observe |
|---|---|---|
| Formatage du NameNode (`hdfs namenode -format`) | [01](docs/screenshots/01_format-namenode_20h11.jpg) | Répertoire `/usr/local/hadoop/data/namenode` formaté. L'arrêt affiché ensuite est normal. |
| Démarrage des démons | [02](docs/screenshots/02_start-daemons_20h13.jpg) | `hdfs --daemon start namenode` puis `datanode`. |
| Premier contrôle | [03](docs/screenshots/03_jps-sans-datanode_20h14.jpg) | `jps` sans DataNode et `dfsadmin -report` à 0 B : aucun DataNode connecté. |
| Reformatage puis redémarrage | [05](docs/screenshots/05_reformat-namenode_20h19.jpg), [06](docs/screenshots/06_restart-daemons_20h20.jpg) | Nouveau formatage du NameNode (nouveau BlockPoolId). |
| Correction | [07](docs/screenshots/07_fix-datanode_20h23.jpg) | Le DataNode ne démarre pas. Après arrêt du NameNode, `rm -rf /usr/local/hadoop/data/datanode/*` puis redémarrage : `jps` affiche NameNode et DataNode. |
| Cluster opérationnel | [08](docs/screenshots/08_dfsadmin-report-ok_20h25.jpg) | `Live datanodes (1)`, capacité 1006,85 Go, 0 bloc. |

> **Leçon retenue :** après un `namenode -format`, le NameNode reçoit un nouvel identifiant de cluster que le DataNode refuse s'il garde l'ancien. La correction appliquée est de vider le répertoire du DataNode (cause habituelle, le journal du DataNode n'a pas été consulté).

### 2. Préparation du jeu de données

| Étape | Capture | Ce qu'on observe |
|---|---|---|
| Décompression | [09](docs/screenshots/09_unzip-dataset_20h34.jpg) | `unzip online+retail+ii.zip` donne `online_retail_II.xlsx`. |
| Incident pip | [04](docs/screenshots/04_erreur-pip_20h14.jpg) | `externally-managed-environment` (PEP 668) : Ubuntu protège le Python du système. |
| Installation | [10](docs/screenshots/10_pip-install_20h37.jpg) | `pip install pandas openpyxl --break-system-packages` (un environnement virtuel est l'alternative recommandée). |
| Excel vers CSV | [11](docs/screenshots/11_xlsx-vers-csv_20h40.jpg) | Toutes les feuilles concaténées avec pandas : `online_retail.csv` (91 Mo). |
| Gros fichier | [12](docs/screenshots/12_gros-retail_20h42.jpg) | Contenu répété 4 fois : `gros_retail.csv` (362 Mo), supérieur à un bloc HDFS de 128 Mo. |

### 3. Dépôt et lecture dans HDFS

| Étape | Capture | Ce qu'on observe |
|---|---|---|
| Dépôt | [13](docs/screenshots/13_hdfs-put-ls_20h44.jpg) | `-mkdir -p`, `-put`, `-ls -h` : 361,8 Mo, facteur de réplication 2. |
| Lecture | [14](docs/screenshots/14_hdfs-cat-head_20h45.jpg) | `-cat … \| head -n 5` affiche l'en-tête et 4 lignes. Le message « Unable to write to output stream » est dû à `head`. |
| Incident YARN | [15](docs/screenshots/15_erreur-yarn_20h48.jpg) | Connexion au ResourceManager (`0.0.0.0:8032`) impossible, interrompue par Ctrl+C. Hors périmètre HDFS. |

### 4. Blocs et tolérance aux pannes

| Étape | Capture | Ce qu'on observe |
|---|---|---|
| `hdfs fsck` | [16](docs/screenshots/16_fsck_20h49.jpg) | HEALTHY, 379 400 585 octets, **3 blocs**. Réplication par défaut 2 mais un seul DataNode : 3 blocs under-replicated, réplication moyenne 1,0. |
| Panne du DataNode | [17](docs/screenshots/17_panne-datanode_20h51.jpg) | `BlockMissingException` : « No live nodes contain current block ». Une seule copie, donc fichier illisible. |
| `fsck` après la panne | [18](docs/screenshots/18_fsck-apres-panne_20h51.jpg) | Toujours HEALTHY : fsck s'appuie sur les métadonnées du NameNode, qui met du temps à déclarer le DataNode mort. |
| Notes pour 2 DataNodes | [19](docs/screenshots/19_notes-2-datanodes.jpg) | Test prévu avec un second DataNode (`/opt/hadoop/hdfs/datanode2`) : avec 2 copies, la lecture réussirait. **Non réalisé** pendant ce TP. |

## Bilan

- HDFS local fonctionnel, fichier de 362 Mo découpé en **3 blocs**.
- Avec un seul DataNode, la réplication à 2 ne peut pas être satisfaite et la panne du nœud rend les données inaccessibles.
- Pour démontrer la tolérance aux pannes, il faut au moins 2 DataNodes.

## Reproduire

```bash
bash scripts/01_start_hdfs.sh
bash scripts/02_prepare_dataset.sh   # place d'abord online+retail+ii.zip dans le dossier courant
bash scripts/03_load_hdfs.sh
bash scripts/04_inspect_and_fault_test.sh
```
# TP-HDFS-INTRO-TO-BIG-DATA
