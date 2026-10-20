
<p align="center">
  <img src="https://img.shields.io/badge/Python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python 3.11" />
  <img src="https://img.shields.io/badge/Jupyter-Notebook-F37626?style=for-the-badge&logo=jupyter&logoColor=white" alt="Jupyter Notebook" />
  <img src="https://img.shields.io/badge/uv-Environment-6C3FB5?style=for-the-badge&logo=astral&logoColor=white" alt="uv" />
  <img src="https://img.shields.io/badge/Google-Gemini--API-8E75B2?style=for-the-badge&logo=googlegemini&logoColor=white" alt="Google Gemini API" />
  <img src="https://img.shields.io/badge/LangChain-RAG-1C3C3C?style=for-the-badge&logo=chainlink&logoColor=white" alt="LangChain" />
  <img src="https://img.shields.io/badge/ChromaDB-Vector--Database-FF6F61?style=for-the-badge" alt="ChromaDB" />
  <img src="https://img.shields.io/badge/PyPDF-Document--Loading-DC382D?style=for-the-badge" alt="PyPDF" />
  <img src="https://img.shields.io/badge/LLM-Judge--Evaluation-0A7B83?style=for-the-badge" alt="LLM Judge Evaluation" />
  <img src="https://img.shields.io/badge/Status-Terminé-success?style=for-the-badge" alt="Status" />
</p>

---


# TP4 DSD — RAG sur CV

Application **RAG** (*Retrieval-Augmented Generation*) qui répond à des questions sur mon CV à partir de son contenu
uniquement, avec une évaluation des réponses par un LLM juge et une expérience sur le nombre de chunks récupérés (`k`).

![Architecture RAG + juge](rag_juge.gif)

## Pipeline

| Étape | Outil | Détail |
| :--- | :--- | :--- |
| Chargement | `PyPDFLoader` | CV d'une page, 716 tokens |
| Chunking | `RecursiveCharacterTextSplitter` (tiktoken) | 100 tokens, overlap 20 → 9 chunks |
| Embeddings | `gemini-embedding-001` | vecteurs de dimension 3072 |
| Stockage | ChromaDB | collection `cv_mouad` |
| Retrieval | recherche par similarité (Chroma) | `k = 3` |
| Génération | `gemini-3.5-flash-lite` | réponse uniquement depuis le contexte, sinon « Je ne sais pas! » |
| Évaluation | `gemini-3.5-flash` (juge) | groundedness notée de 1 à 5 |

## Résultats principaux

- **4/4** questions factuelles correctes (FlowGuard, JADE, bases de données, accuracy 99,10 %), toutes notées **5/5**.
- **« Je ne sais pas! »** pour les questions hors CV (niveau d'anglais, Coupe du monde 2022) : pas d'hallucination.
- **Expérience sur `k`** avec *« Combien de projets dans le CV ? »* :
  - `k = 1` → 3 projets sur 5 (réponse incomplète) ;
  - `k = 3` → « 6 projets » (doublon causé par l'overlap et l'ordre des chunks) ;
  - le juge note **5/5 dans les deux cas** : la groundedness ne mesure ni la complétude ni l'exactitude.

## Détails et analyse : [`REPONSES.md`](REPONSES.md).

### 1. Document et objectif

J'ai appliqué le notebook `rag.ipynb` à **mon CV** (`pdfs/CV.pdf`, 1 page). Le but est que le LLM réponde aux questions
en s'appuyant uniquement sur les passages du CV que le retriever lui fournit, et réponde « Je ne sais pas! » sinon.

### 2. Modèles utilisés

Pas de clé OpenAI : j'ai utilisé l'API gratuite **Google Gemini**, comme l'autorise la consigne.

- RAG : `gemini-3.5-flash-lite` (à la place de `gpt-4o-mini`)
- Juge : `gemini-3.5-flash`, plus puissant que le modèle du RAG (à la place de `gpt-4o`)
- Embeddings : `gemini-embedding-001` (à la place de `text-embedding-3-small`)

Adaptations du code : `.text` au lieu de `.content` (les réponses Gemini sont des listes de blocs) et
`langchain_chroma` au lieu de `langchain_community.vectorstores` (déprécié). La clé est dans `.env`, exclu du dépôt.

### 3. Chunking

- Le CV fait **716 tokens**. Avec 300 tokens par chunk (paramètre du notebook fourni), on n'aurait que **3 chunks**, et le
  retriever renverrait presque tout le CV.
- J'ai choisi **100 tokens, overlap 20 → 9 chunks**. Chaque chunk couvre une partie : coordonnées, domaines de
  compétence, 1 à 3 projets, formation, compétences techniques.
- **Extraction du PDF :** `pypdf` colle des mots séparés uniquement par la mise en page (*« temps réelProjet »*,
  *« ProgrammationPython »*) et coupe des mots en fin de ligne (*« Infor- matique »*). Le RAG reste fonctionnel, mais
  c'est du bruit dans les chunks.

### 4. Retriever

Dans le notebook fourni, `search_Kwargs={'k',5}` ne fonctionne pas :
- `search_Kwargs` (avec un K majuscule) n'est pas le bon nom de paramètre, il faut `search_kwargs` ;
- `{'k',5}` est un ensemble (*set*), pas un dictionnaire `{'k': 5}`.

Le paramètre est ignoré et la valeur par défaut `k=4` est utilisée. J'utilise `search_kwargs={"k": 3}`.
Test : pour *« Quelle accuracy le modèle de détection des feux de forêt a-t-il obtenue ? »*, le chunk 3 (celui du projet)
arrive en premier.

### 5. Réponses du RAG

| Question | Réponse | Juste ? |
| :--- | :--- | :---: |
| Technologies du projet FlowGuard ? | Kafka, scoring ML / apprentissage fédéré FedAvg, Neo4j, Dash/Plotly | ✅ |
| Framework du système multi-agents bataille navale ? | JADE | ✅ |
| Bases de données maîtrisées ? | MySQL, Oracle Database, Neo4j, Apache Cassandra | ✅ |
| Accuracy du modèle de détection des feux de forêt ? | 99,10 % | ✅ |
| Niveau d'anglais ? *(absent du CV)* | Je ne sais pas! | ✅ |
| Vainqueur de la Coupe du monde 2022 ? *(hors sujet)* | Je ne sais pas! | ✅ |

Le CV n'a pas de section « Langues » : le modèle ne déduit pas un niveau d'anglais à partir du reste du CV. Pour la
Coupe du monde, il connaît la réponse mais respecte la consigne du prompt.

### 6. Évaluation par le juge

Prompt du juge du notebook fourni, avec une ligne finale imposée `NOTE = x/5` pour extraire la note automatiquement.
Les réponses du §5 sont réutilisées : le RAG n'est pas rappelé une deuxième fois pour l'évaluation.

| Question | Note |
| :--- | :---: |
| FlowGuard | 5/5 |
| Bataille navale (JADE) | 5/5 |
| Bases de données | 5/5 |
| Accuracy feux de forêt | 5/5 |
| Niveau d'anglais (« Je ne sais pas! ») | 5/5 |

### 7. Expérience : influence de `k` sur une question de synthèse

Question : *« Combien de projets sont présentés dans le CV et lesquels ? »* (réponse attendue : **5**, répartis sur les
chunks 2 à 5).

| | Chunks récupérés | Réponse du RAG | Note du juge |
| :--- | :--- | :--- | :---: |
| `k = 1` | 4 | 3 projets (Gestion médicale, Bataille navale, Odoo) | 5/5 |
| `k = 3` | 4, 3, 2 | « 6 projets » : *Gestion médicale distribuée* compté deux fois | 5/5 |

**Analyse :**
- Avec `k = 1`, la réponse est **fidèle mais incomplète** : FlowGuard et les feux de forêt sont dans d'autres chunks.
- Avec `k = 3`, toute l'information est présente mais la réponse est **fausse** :
  1. à cause de l'**overlap**, le titre *« Gestion médicale distribuée »* est à la fois en fin de chunk 3 et en début de
     chunk 4, il apparaît deux fois dans le contexte ;
  2. les chunks sont concaténés **par ordre de similarité (4, 3, 2)** et non dans l'ordre du document : une puce de fin de
     chunk se retrouve collée au titre du chunk suivant, d'où des attributions erronées (le dashboard Dash/Plotly de
     FlowGuard est rattaché au projet Odoo).
- **Le juge met 5/5 aux deux réponses.** La métrique de *groundedness* vérifie que la réponse vient du contexte ; elle ne
  détecte ni une réponse incomplète ni un comptage faux.

**Améliorations possibles :**
- trier les chunks récupérés selon leur position dans le document et retirer les doublons d'overlap avant de construire
  le contexte ;
- découper le CV par section (un chunk par projet) au lieu d'un nombre fixe de tokens ;
- ajouter des métriques de **complétude** et d'**exactitude** (comparaison à une réponse de référence) en plus de la
  groundedness.

### 8. Conclusion

Le RAG est fiable pour les questions **factuelles** qui portent sur un passage précis du CV, et ne répond pas aux
questions hors document. Les questions de **synthèse** sont plus difficiles : elles dépendent de `k`, de l'overlap et de
l'ordre des chunks. Enfin, une bonne note de groundedness ne garantit pas une bonne réponse : l'évaluation d'un RAG doit
combiner plusieurs métriques.

### Captures d'écran

| | |
| :--- | :--- |
| Connexion à Gemini | ![](captures/01_connexion_gemini.png) |
| Chargement et chunking | ![](captures/02_chunking.png) |
| Embeddings et ChromaDB | ![](captures/03_embeddings_chromadb.png) |
| Retriever | ![](captures/04_retriever.png) |
| Prompt et réponse du LLM | ![](captures/05_prompt_et_reponse.png) |
| Réponses du RAG | ![](captures/06_reponses_rag.png) |
| Évaluation détaillée par le juge | ![](captures/07_juge_detail.png) |
| Tableau des notes | ![](captures/08_tableau_notes.png) |
| Expérience sur `k` | ![](captures/09_experience_k.png) |
| Analyse | ![](captures/10_analyse_k.png) |

## Contenu

```
rag_cv.ipynb   notebook exécuté
REPONSES.md    réponses, analyse et captures d'écran
captures/      captures d'écran de l'exécution
pdfs/CV.pdf    document interrogé
pyproject.toml / uv.lock   dépendances (uv)
```

## Exécution

```powershell
uv sync
echo GOOGLE_API_KEY=votre_cle > .env   # clé gratuite : https://aistudio.google.com
```

Ouvrir `rag_cv.ipynb`, sélectionner le kernel `.venv`, puis *Run All*.
Le palier gratuit de Gemini est limité à environ 20 requêtes par jour et par modèle ; le notebook en utilise une dizaine
par modèle.

---

<p align="center">
  <strong>Réalisé par : Aguelloul Mouad</strong><br>
  <strong>Encadré par : Lhoussaine Daoudi</strong>
</p>
