# TP4 DSD — RAG sur CV

**Aguelloul Mouad** · Master IA & Digital Computing, FST Béni Mellal · Module *Système Distribué Agentique*

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

Détails et analyse : [`REPONSES.md`](REPONSES.md).

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
