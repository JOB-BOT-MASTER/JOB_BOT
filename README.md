# JOB BOT - Bot Discord de recherche d'emploi

Bot Discord permettant de rechercher des offres d'emploi et de générer des lettres de motivation personnalisées à partir d'un CV en PDF.

## Fonctionnement global

L'utilisateur envoie la commande `!search_job` sur Discord avec son CV en PDF en pièce jointe. Le bot effectue alors les étapes suivantes :

1. Extraction et restructuration du texte du CV (Groupe 4)
2. Scraping des offres d'emploi sur LinkedIn et Welcome to the Jungle (Groupes 2 et 3)
3. Analyse sémantique pour trouver les offres les plus pertinentes (Groupe 5)
4. Génération d'une lettre de motivation personnalisée en LaTeX pour chaque offre (Groupe 5)
5. Envoi des résultats à l'utilisateur sur Discord (Groupe 1)

## Structure du projet

```
JOB_BOT/
├── bot.py                        # Point d'entrée principal (Groupe 1)
├── requirements.txt              # Packages Python nécessaires
├── .env                          # Variables secrètes (non commité)
├── env.example                   # Template pour le .env
├── scraper/
│   ├── scraper_site1.py          # Scraping LinkedIn (Groupe 2)
│   └── scraper_site2.py          # Scraping Welcome to the Jungle (Groupe 3)
├── cv_parser/
│   └── pdf_parser.py             # Extraction et restructuration du CV (Groupe 4)
├── llm_handler/
│   ├── embeddings.py             # Analyse sémantique (Groupe 5)
│   ├── generator.py              # Génération de lettres de motivation (Groupe 5)
│   └── config.py                 # Configuration du module LLM (Groupe 5)
└── data/
    └── offers.csv                # Fichier intermédiaire des offres scrapées
```

## Groupes

| Groupe | Rôle | Fichier(s) |
|--------|------|------------|
| Groupe 1 | Bot Discord et coordination | `bot.py` |
| Groupe 2 | Scraping LinkedIn | `scraper/scraper_site1.py` |
| Groupe 3 | Scraping Welcome to the Jungle | `scraper/scraper_site2.py` |
| Groupe 4 | Extraction et restructuration du CV | `cv_parser/pdf_parser.py` |
| Groupe 5 | Analyse sémantique et génération de lettres | `llm_handler/` |

## Installation

### Prérequis

- Python 3.11
- Google Chrome installé
- Tesseract OCR installé : https://github.com/UB-Mannheim/tesseract/wiki

### Étapes

1. Cloner le repo :
```bash
git clone https://github.com/JOB-BOT-MASTER/JOB_BOT.git
cd JOB_BOT
```

2. Créer et activer l'environnement virtuel :
```bash
# Windows
python -m venv venv
venv\Scripts\activate

# Mac/Linux
python -m venv venv
source venv/bin/activate
```

3. Installer les packages :
```bash
pip install -r requirements.txt
```

4. Configurer les variables d'environnement :
   - Copier `env.example` en `.env`
   - Remplir les valeurs suivantes :
```
DISCORD_TOKEN=votre_token_discord
GEMINI_API_KEY=votre_cle_gemini
```

5. Lancer le bot :
```bash
python bot.py

## Utilisation

Dans un salon Discord, envoyer la commande suivante avec le CV en PDF en pièce jointe :

```
!search_job --type Data Science --loc Strasbourg
```

Le bot retournera pour chaque offre pertinente :
- Le titre du poste, l'entreprise et la localisation
- Un score de pertinence par rapport au CV
- Un lien vers l'offre
- Un fichier `.tex` contenant la lettre de motivation (à compiler sur Overleaf)

### Autres commandes

| Commande | Description |
|----------|-------------|
| `!aide` | Affiche l'aide du bot |
| `!ping` | Vérifie que le bot est en ligne |

## Ouvrir la lettre de motivation

La lettre de motivation est générée au format LaTeX (fichier `.tex`). Pour l'ouvrir en PDF :

1. Télécharger le fichier `.tex` envoyé par le bot
2. Aller sur https://overleaf.com
3. Créer un nouveau projet et coller le contenu du fichier
4. Compiler pour obtenir le PDF final

## Règles GitHub

- Ne jamais commiter le fichier `.env`
- Ne jamais commiter le dossier `venv/`
- Toujours faire `git pull` avant de commencer à coder
- Mettre à jour `requirements.txt` après avoir installé un nouveau package et prévenir les autres sur Discord

```
