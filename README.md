
# Installation requise:
# pip install google-genai google-api-python-client

import os
from google import genai
from google.genai import types
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

# --- CONFIGURATION DES CLÉS API ---
# Il est recommandé d'utiliser des variables d'environnement pour la sécurité
GEMINI_API_KEY = os.environ.get("GEMINI_API_KEY", "AIzaSyAsqliryxIASl-CIaZTr0DeJZsK2_M_S3A")
YOUTUBE_API_KEY = os.environ.get("YOUTUBE_API_KEY", "AIzaSyDPhUEzAK7Z3FPStKijhamlt9KPRKqKuWk")

# --- RÉCUPÉRATION DES COMMENTAIRES YOUTUBE ---

def get_youtube_comments(video_id: str, max_results: int = 100):
    """
    Récupère les commentaires d'une vidéo YouTube via l'API YouTube Data v3.
    
    Args:
        video_id: L'identifiant de la vidéo YouTube
        max_results: Nombre maximum de commentaires à récupérer (max 100 par requête)
    
    Returns:
        Tuple contenant (texte_commentaires, liste_commentaires_detailles) ou (message_erreur, None)
    """
    print(f"Récupération des commentaires pour la vidéo : {video_id}...")
    
    try:
        # Créer le client YouTube API
        youtube = build('youtube', 'v3', developerKey=YOUTUBE_API_KEY)
        
        # Récupérer les commentaires (threads de commentaires)
        request = youtube.commentThreads().list(
            part="snippet",
            videoId=video_id,
            maxResults=max_results,
            order="relevance",  # Commentaires les plus pertinents
            textFormat="plainText"
        )
        
        response = request.execute()
        
        # Extraire le texte et les détails des commentaires
        comments = []
        detailed_comments = []
        
        for item in response.get('items', []):
            snippet = item['snippet']['topLevelComment']['snippet']
            comment_text = snippet['textDisplay']
            like_count = snippet.get('likeCount', 0)
            author = snippet.get('authorDisplayName', 'Anonyme')
            
            comments.append(comment_text)
            detailed_comments.append({
                'text': comment_text,
                'likes': like_count,
                'author': author
            })
        
        if not comments:
            return "Aucun commentaire trouvé pour cette vidéo.", None
        
        print(f"✓ {len(comments)} commentaires récupérés avec succès.")
        return "\n".join(comments), detailed_comments
    
    except HttpError as e:
        error_message = f"Erreur HTTP {e.resp.status}: {e.error_details}"
        print(f"✗ {error_message}")
        
        if e.resp.status == 403:
            return "Erreur: Les commentaires sont désactivés pour cette vidéo ou quota API dépassé.", None
        elif e.resp.status == 404:
            return "Erreur: Vidéo introuvable. Vérifiez l'ID de la vidéo.", None
        else:
            return f"Erreur lors de la récupération des commentaires: {error_message}", None
    
    except Exception as e:
        print(f"✗ Erreur inattendue: {str(e)}")
        return f"Erreur inattendue: {str(e)}", None


# --- EXTRACTION DE L'ID DEPUIS UNE URL ---

def extract_video_id(url_or_id: str) -> str:
    """
    Extrait l'ID de la vidéo depuis une URL YouTube ou retourne l'ID si déjà fourni.
    
    Supporte les formats:
    - https://www.youtube.com/watch?v=VIDEO_ID
    - https://youtu.be/VIDEO_ID
    - VIDEO_ID directement
    """
    if "youtube.com/watch?v=" in url_or_id:
        return url_or_id.split("watch?v=")[1].split("&")[0]
    elif "youtu.be/" in url_or_id:
        return url_or_id.split("youtu.be/")[1].split("?")[0]
    else:
        return url_or_id


# --- ANALYSE PAR GEMINI ---

def display_top_comments(detailed_comments: list, client, model: str):
    """
    Sélectionne et affiche les 5 meilleurs commentaires selon Gemini.
    
    Args:
        detailed_comments: Liste de dictionnaires contenant les commentaires avec leurs détails
        client: Client Gemini initialisé
        model: Nom du modèle Gemini à utiliser
    """
    print(f"\n{'='*60}")
    print("🏆 SÉLECTION DES 5 MEILLEURS COMMENTAIRES")
    print(f"{'='*60}\n")
    
    # Préparer la liste des commentaires pour Gemini
    comments_list = "\n\n".join([
        f"[Commentaire {i+1}] (👍 {c['likes']} likes)\n{c['text']}"
        for i, c in enumerate(detailed_comments)
    ])
    
    prompt = f"""
Parmi les commentaires suivants, sélectionne les 5 MEILLEURS commentaires selon ces critères:
1. Pertinence et utilité du contenu
2. Détails et profondeur de l'analyse
3. Constructivité et valeur ajoutée
4. Nombre de likes (indicateur de qualité communautaire)

IMPORTANT: 
- Retourne UNIQUEMENT les commentaires dans leur LANGUE ORIGINALE, sans traduction
- Numérote-les de 1 à 5
- Pour chaque commentaire, indique brièvement (en français) pourquoi il a été sélectionné
- Conserve EXACTEMENT le texte original du commentaire

Format de réponse:
**1. [Raison de sélection en français]**
[Texte du commentaire dans sa langue originale]

**2. [Raison de sélection en français]**
[Texte du commentaire dans sa langue originale]

... etc.

Commentaires disponibles:
---
{comments_list}
---
"""
    
    try:
        response = client.models.generate_content(
            model=model,
            contents=[
                types.Content(
                    role="user",
                    parts=[types.Part.from_text(text=prompt)]
                )
            ]
        )
        
        print(response.text)
        print(f"\n{'='*60}\n")
        
    except Exception as e:
        print(f"✗ Erreur lors de la sélection des meilleurs commentaires : {e}\n")


def analyze_video_usefulness(video_id: str, max_comments: int = 100):
    """
    Analyse les commentaires d'une vidéo YouTube pour déterminer sa pertinence/utilité.
    
    Args:
        video_id: L'identifiant de la vidéo YouTube (ou URL complète)
        max_comments: Nombre maximum de commentaires à analyser
    """
    # Extraire l'ID si une URL est fournie
    video_id = extract_video_id(video_id)
    
    print(f"\n{'='*60}")
    print(f"ANALYSE DE LA VIDÉO: {video_id}")
    print(f"{'='*60}\n")
    
    # Récupérer les commentaires
    comments, detailed_comments = get_youtube_comments(video_id, max_comments)
    
    if detailed_comments is None or "Erreur" in comments or "Aucun commentaire" in comments:
        print(f"\n⚠️  {comments}")
        return
    
    # Configuration du client Gemini
    try:
        client = genai.Client(api_key=GEMINI_API_KEY)
    except Exception as e:
        print(f"\n✗ Erreur lors de l'initialisation du client Gemini : {e}")
        return
    
    # Modèle optimisé pour l'analyse de texte
    model = "gemini-2.0-flash-exp"
    
    # Prompt détaillé pour l'analyse
    prompt = f"""
Analyse les commentaires suivants d'une vidéo YouTube et détermine son niveau de PERTINENCE et d'UTILITÉ.

Évalue les aspects suivants:
1. **Sentiment global** : Positif, négatif ou neutre
2. **Qualité du contenu** : Clarté, utilité, profondeur
3. **Satisfaction des spectateurs** : Recommandations, remerciements
4. **Points négatifs** : Critiques constructives ou problèmes signalés

Fournis ton analyse dans ce format:

**Score de Pertinence: X/10**

**Verdict: [TRÈS PERTINENTE | MODÉRÉMENT PERTINENTE | PEU PERTINENTE]**

**Analyse détaillée:**
- Sentiment global: [description]
- Points forts: [liste]
- Points faibles: [liste]
- Recommandation: [conclusion]

**Commentaires analysés:**
---
{comments}
---
"""
    
    # Génération de l'analyse
    print("📊 Analyse en cours avec Gemini...\n")
    try:
        response = client.models.generate_content(
            model=model,
            contents=[
                types.Content(
                    role="user",
                    parts=[types.Part.from_text(text=prompt)]
                )
            ]
        )
        
        print(f"\n{'='*60}")
        print("RÉSULTAT DE L'ANALYSE")
        print(f"{'='*60}\n")
        print(response.text)
        print(f"\n{'='*60}\n")
        
        # Extraire et afficher les 5 meilleurs commentaires
        display_top_comments(detailed_comments, client, model)
        
    except Exception as e:
        print(f"\n✗ Erreur lors de l'appel à l'API Gemini : {e}")


# --- FONCTION PRINCIPALE ---

def main():
    """
    Point d'entrée principal du programme.
    """
    print("\n" + "="*60)
    print("ANALYSEUR DE PERTINENCE DE VIDÉOS YOUTUBE")
    print("="*60 + "\n")
    
    # Exemples d'utilisation:
    
    # Option 1: Utiliser un ID de vidéo directement
    # video_id = "dQw4w9WgXcQ"
    
    # Option 2: Utiliser une URL complète
    # video_id = "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
    
    # Option 3: Demander à l'utilisateur
    video_id = input("Entrez l'URL ou l'ID de la vidéo YouTube à analyser: ").strip()
    
    if not video_id:
        print("⚠️  Aucune vidéo spécifiée. Utilisation d'un exemple.")
        # Remplacez par un vrai ID de vidéo pour tester
        video_id = "dQw4w9WgXcQ"
    
    # Lancer l'analyse
    analyze_video_usefulness(video_id, max_comments=100)


if __name__ == "__main__":
    main()
