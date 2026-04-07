# Groupe 4 : Module d'Extraction de texte à partir d'un CV

**Projet :** JOB_BOT Bot Discord d'aide à la recherche d'emploi

**UE :** Conduite de Projet-Master 1 DS2E-Université de Strasbourg

**Membres :** Alpha Oumar **DIALLO**-Rayhana **Ben HIM**-Antoine **PACCHIONI**-Yaye Fatou **GNINGUE**

## 1. Objectif du module

Le Groupe 4, nous sommes responsable de l'extraction textuelle des CV soumis par les utilisateurs au format PDF. Le module prend en entrée un fichier PDF (quel que soit son mode de création) et retourne en sortie un texte restructuré par rubriques, directement exploitable par le module d'analyse du Groupe 5.

<img width="1129" height="314" alt="image" src="https://github.com/user-attachments/assets/c06b15b9-63c8-4919-9b6e-99f4bda2d29b" />


Le script produit une unique fonction importable, extraire_cv(chemin_pdf), conçue pour être appelée depuis le bot Discord du Groupe 1 sans dépendance au reste du code.

## 2. Problématique technique

L'extraction de texte depuis un PDF est triviale sur des documents à mise en page linéaire (rapports, articles). Elle devient problématique sur des CV pour trois raisons identifiées lors de nos tests.

**Les colonnes multiples.** La majorité des CV modernes utilisent deux colonnes (compétences à gauche, formations et expériences à droite). Un parser qui lit le PDF dans son ordre interne mélange les contenus des deux colonnes sur une même ligne, produisant un texte incohérent.

**Les CV sans calque texte.** Les CV créés sur des plateformes graphiques (Canva, Photoshop) ou scannés depuis un document papier sont exportés en PDF sous forme d'image. Aucune donnée textuelle n'est accessible par extraction classique : le parser retourne une chaîne vide.

**La fragmentation des rubriques.** Même avec un parser performant, les titres de section ("Formations", "Compétences") se retrouvent parfois séparés de leur contenu à cause du découpage en blocs textuels du PDF.

## 3. Choix techniques et justifications

### 3.1 PyMuPDF (fitz) plutôt que PyPDF2

Nous avons initialement testé PyPDF2, la bibliothèque la plus répandue pour la lecture de PDF en Python. Le résultat sur notre propre CV (format deux colonnes) était inutilisable : les rubriques "Langues" et "Compétences" (colonne gauche) se retrouvaient intercalées avec les "Formations" (colonne droite).

Exemple de sortie PyPDF2 sur le CV d'Alpha Oumar DIALLO :

```
Rédaction, Présentation, Master 1 Data Science pour l'Economie Reporting
Word Excel(TCD, Licence Economie, Statistique et Modélisation Fonctions,
Dashboard) De septembre 2022 à juin 2024 Powerpoint Université de Lille...
```

**Cause identifiée :** PyPDF2 lit les objets texte dans l'ordre d'écriture interne du fichier PDF, qui ne correspond pas à l'ordre visuel de lecture. PyMuPDF, en revanche, utilise les coordonnées spatiales (x, y) de chaque bloc texte pour reconstituer l'ordre de lecture naturel. Il est par ailleurs significativement plus rapide, son moteur de rendu (MuPDF) étant écrit en C.

### 3.2 Tesseract OCR comme mécanisme de fallback

Pour gérer les CV exportés sous forme d'image, nous avons intégré un mécanisme de détection automatique suivi d'un traitement OCR.

**Critère de déclenchement :** Si le texte extrait par PyMuPDF sur une page contient moins de 50 caractères, la page est considérée comme une image. Ce seuil a été choisi empiriquement : une page de CV standard contient entre 500 et 3000 caractères ; un seuil de 50 laisse une marge pour les éléments résiduels (numéros de page, pieds de page) tout en détectant les pages effectivement vides de texte.

**Processus de fallback :**

1. La page est convertie en image bitmap à 300 DPI via PyMuPDF. La résolution de 300 DPI a été retenue car elle offre un taux de reconnaissance supérieur à 95% avec Tesseract, contre environ 80% à 150 DPI. Une résolution de 600 DPI n'apporte pas de gain significatif mais double le temps de traitement.

2. L'image est convertie au format PIL via la bibliothèque Pillow pour compatibilité avec Tesseract.

3. Tesseract effectue la reconnaissance optique avec les modèles de langue français et anglais (lang='fra+eng'), les CV contenant fréquemment des termes techniques anglais.

**Choix de Tesseract plutôt qu'une API cloud (Google Vision, AWS Textract) :** Tesseract est gratuit, open source, et fonctionne hors ligne. Dans le cadre d'un projet académique sans budget, c'est le choix le plus pragmatique. L'inconvénient est qu'il nécessite une installation système sur chaque machine qui fais tourner le code.

### 3.3 Gemini 2.5 Flash pour la restructuration

Après l'extraction (PyMuPDF ou OCR), le texte obtenu est souvent fragmenté : les rubriques sont détachées de leur contenu, des informations de colonnes différentes se retrouvent adjacentes. Nous utilisons l'API Gemini pour remettre en ordre ces fragments.

**Point essentiel : le LLM ne fait pas d'analyse.** Son rôle est strictement limité à la restructuration. Le prompt impose trois contraintes explicites : ne rien résumer, ne rien inventer, regrouper les titres avec leurs contenus. L'analyse sémantique (comparaison CV/offre, génération de lettre) est la responsabilité exclusive du Groupe 5.

**Choix du modèle Flash :** Pour une tâche de reformatage (pas de raisonnement complexe), le modèle Flash offre un temps de réponse de 2 à 3 secondes contre 10 à 15 pour les modèles plus lourds, pour un résultat équivalent sur ce type de tâche.

## 4. Code source commenté (extraits clés)

Le script complet est dans cv_parser.py. Nous détaillons ici les passages qui méritent des explications.

### 4.1 Extraction et détection automatique

```python
document = fitz.open(chemin_pdf)
texte_brut = ""

for page_num, page in enumerate(document):
    texte_page = page.get_text("text")

    # Détection de page image : seuil empirique de 50 caractères
    if len(texte_page.strip()) < 50:
        pixmap = page.get_pixmap(dpi=300)  # Conversion en image HD
        image_pil = Image.frombytes("RGB", [pixmap.width, pixmap.height], pixmap.samples)
        texte_page = pytesseract.image_to_string(image_pil, lang='fra+eng')

    texte_brut += texte_page + "\n"
```

La boucle traite chaque page indépendamment. Le test len(texte_page.strip()) < 50 agit comme un aiguillage : texte suffisant → on garde l'extraction PyMuPDF ; texte insuffisant → on bascule sur l'OCR. Ce mécanisme est transparent pour l'appelant.

### 4.2 Prompt de restructuration

```python
prompt = f"""
Voici le texte brut extrait d'un CV au format PDF. À cause des colonnes,
le texte et les rubriques sont parfois mélangés ou coupés au mauvais endroit.

Ton rôle est de RESTRUCTURER ce texte de manière parfaitement logique et lisible.
Règles absolues :
- NE RÉSUME RIEN.
- N'AJOUTE AUCUNE INFORMATION INVENTÉE.
- Regroupe correctement les titres avec leurs contenus.
- Formate le résultat proprement.

Texte brut à traiter :
{texte_brut}
"""
```

Les contraintes du prompt ont été formulées après plusieurs itérations. Sans la règle "NE RÉSUME RIEN", Gemini condensait les listes de compétences. Sans "N'AJOUTE AUCUNE INFORMATION INVENTÉE", il complétait les niveaux de langue non spécifiés dans le CV.

## 5. Interface de sortie

La fonction retourne systématiquement un dictionnaire à structure fixe, indépendamment du résultat.

**Cas nominal :**

```python
{
    "statut": "succes",
    "texte_original": "texte brut avant restructuration",
    "texte_propre": "texte restructuré par Gemini",
    "erreur": None
}
```

**Cas d'erreur :**

```python
{
    "statut": "erreur",
    "texte_propre": "",
    "erreur": "Le document est illisible ou vide, même après tentative de lecture OCR."
}
```

Le champ texte_original est conservé à des fins de débogage (comparaison brut/propre). Le Groupe 5 consomme exclusivement texte_propre. Le Groupe 1 teste statut pour décider de la réponse Discord.

**Exemple de sortie réelle sur le CV d'Alpha Oumar DIALLO :**

Le CV original (format Canva, deux colonnes, PDF image) a été détecté en OCR à la page 1, puis restructuré par Gemini. Le champ texte_propre contenait l'intégralité des informations correctement regroupées : coordonnées, profil, formations (Master 1 DS2E, Licence Économie Lille, Licence UCAD Dakar), projets académiques, compétences techniques, langues et centres d'intérêt. Sans aucune perte ni invention.

## 6. Intégration avec les autres groupes

```python
# Dans bot.py (Groupe 1)
from cv_parser import extraire_cv

resultat = extraire_cv("temp_cv.pdf")

if resultat["statut"] == "succes":
    texte_pour_groupe5 = resultat["texte_propre"]
else:
    await ctx.send(f"Erreur : {resultat['erreur']}")
```

Le module ne gère pas la réception du fichier depuis Discord (responsabilité du Groupe 1) ni l'analyse du contenu (responsabilité du Groupe 5). Cette séparation permet à chaque groupe de modifier son code sans impacter les autres, à condition de respecter le format d'entrée/sortie défini ci-dessus.

## 7. Installation

**Logiciel système**

Tesseract OCR doit être installé sur la machine hôte :

- Windows : télécharger l'installeur depuis github.com/UB-Mannheim/tesseract/wiki
- Mac : brew install tesseract
- Linux : sudo apt-get install tesseract-ocr

**Dépendances Python**

```bash
pip install PyMuPDF pytesseract Pillow python-dotenv google-genai
```

**Variable d'environnement**

Créer un fichier .env à la racine du projet (fichier présent dans .gitignore) :

```
GEMINI_API_KEY=votre_cle_api
```

## 8. Limites identifiées

**Dépendance à Tesseract en local.** L'installation système est une contrainte de déploiement. Le chemin est actuellement configuré pour Windows. Une solution portable nécessiterait soit une détection automatique de l'OS, soit le recours à une API OCR cloud.

**Dépendance à l'API Gemini.** En cas d'indisponibilité de l'API (panne, quota dépassé, clé expirée), le Bouclier 3 est inopérant. Le texte brut est retourné sans restructuration, ce qui peut dégrader un peu la qualité de l'analyse du Groupe 5. Un fallback retournant le texte brut avec un avertissement serait une amélioration pertinente.

**Seuil de détection OCR fixe.** Le seuil de 50 caractères fonctionne sur les cas testés, mais pourrait produire des faux positifs (CV minimaliste déclenchant l'OCR inutilement) ou des faux négatifs (PDF semi-image contenant juste assez de texte résiduel pour ne pas déclencher l'OCR).

**Traitement page par page indépendant.** (pour des CV avec plus d'un page) Une rubrique à cheval sur deux pages pourrait être fragmentée lors de la restructuration.

**Absence de tests automatisés.** Le module a été validé manuellement sur un échantillon restreint de CV. Un jeu de tests couvrant les cas types (une colonne, deux colonnes, scanné, Canva, multipage) renforcerait la fiabilité.

## 9. Structure des fichiers

```
groupe4/
├── cv_parser.py        # Script principal
├── .env                # Clé API Gemini (non versionné)
├── README.md           # Cette documentation
└── requirements.txt    # Dépendances Python
```
## 10. Résumé de la pipeline globale

Le module fonctionne en 3 étapes simples :

**1. Extraction du texte**  
D’abord, le CV au format PDF est lu avec PyMuPDF afin d’en extraire le texte. Si le document ne contient pas de texte exploitable (cas des CV scannés ou exportés en image), un fallback automatique vers Tesseract OCR est déclenché pour récupérer le contenu.

**2. Regroupement du contenu**  
Le texte obtenu est ensuite regroupé en un seul bloc. À ce stade, il reste souvent désorganisé à cause des colonnes ou de la mise en page du CV, mais il constitue une base complète sur laquelle travailler.

**3. Restructuration du CV**  
Enfin, ce texte brut est envoyé à l’API Gemini, dont le rôle est uniquement de le remettre en forme de manière logique (regroupement des rubriques, ordre de lecture cohérent), sans jamais modifier ni inventer d’informations.

La fonction principale `extraire_texte_cv(chemin_pdf)` retourne alors un dictionnaire contenant le texte original, le texte restructuré, ainsi qu’un statut d’exécution. Ce résultat est directement utilisé par le Groupe 5 afin de générer une lettre de motivation pertinente, suite à l'analyse du texte restructuré.
