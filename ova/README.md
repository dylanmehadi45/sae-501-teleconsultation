# Machines virtuelles (OVA)

Les deux VM du projet (broker Mosquitto et serveur web).

> ⚠️ Les fichiers `.ova` sont **trop lourds pour un dépôt Git classique** (souvent plusieurs Go, et GitHub refuse les fichiers > 100 Mo). Ils sont donc exclus par le `.gitignore`.

Deux options pour les partager :
- **Git LFS** si tu veux vraiment les mettre ici (`git lfs track "*.ova"`), dans la limite du quota ;
- plus simple : un lien de téléchargement (Drive universitaire, etc.) ajouté dans ce fichier.

Avant d'exporter une OVA : vérifie qu'elle **ne contient aucun secret** en clair (mots de passe, clés privées) si le dépôt est public.
