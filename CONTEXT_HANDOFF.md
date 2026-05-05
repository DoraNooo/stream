# Context Handoff — YouTube Clipper (Clip_youtube)

Document à copier dans un autre dépôt pour retrouver le contexte d’infra, de prod et les pièges rencontrés sur ce projet.

---

## 1. Ce que fait l’app

- Interface web **Flask** : extraire un clip YouTube (timecodes), télécharger une **vidéo complète**, montage multi-clips, **captions burn-in** (Whisper / `faster-whisper`), bibliothèque locale IndexedDB + **backup serveur** optionnel.
- Pipeline métier : **yt-dlp** (extraction / téléchargement) + **ffmpeg** (encodage ou burn-in sous-titres).

---

## 2. Stack technique

| Composant | Rôle |
|-----------|------|
| Python 3 + Flask | App web |
| Gunicorn | Prod WSGI (**1 worker, plusieurs threads** — voir §8) |
| Nginx | Reverse proxy HTTPS + timeouts SSE |
| systemd | Service `clip-youtube`, fichier env `/etc/clip-youtube/env` |
| yt-dlp + plugins | Extraction YouTube ; `yt-dlp-ejs` pour défis JS ; cookies fichiers |
| Node.js | Runtime JS pour yt-dlp (n-challenge) ; variable `YT_DLP_NODE_PATH` |
| Docker `brainicism/bgutil-ytdlp-pot-provider` | PO Token local (`127.0.0.1:4416`) sur VPS datacenter |
| aria2 (`aria2c`) | Optionnel mais fortement recommandé : téléchargements parallèles |
| faster-whisper | Captions ; CPU ; premier run télécharge les modèles |

---

## 3. Variables d’environnement (résumé)

Fichier type : `/etc/clip-youtube/env` (une ligne par variable ; valeurs avec espaces → **guillemets doubles**).

| Variable | Rôle |
|----------|------|
| `YT_CLIPPER_BACKUP_SECRET` | Active la bibliothèque serveur ; header `X-Backup-Key` pour les routes `/backup*` ; session Flask pour `/auth` et auto-save |
| `YT_DLP_COOKIES_FILE` | Chemin fichier cookies Netscape YouTube (souvent requis sur IP datacenter) |
| `YT_DLP_NODE_PATH` | Ex. `/usr/bin/node` — évite « JS runtimes: none » sous systemd |
| `YT_DLP_VERBOSE` | `1` pour logs yt-dlp dans journalctl |

Après modification du fichier env :

```bash
sudo systemctl daemon-reload   # si le unit change
sudo systemctl restart clip-youtube
```

---

## 4. Déploiement VPS (schéma reproductible)

1. Utilisateur dédié (ex. `clipper`), dépôt dans `~/Clip_youtube`, `venv`, `pip install -r requirements.txt`.
2. **Gunicorn** lié à `127.0.0.1:5000`, timeouts **longs** (clips longs, Whisper, SSE).
3. **Nginx** : `server_name` du sous-domaine ; `proxy_pass` vers le port local ; pour SSE : **`proxy_buffering off`**, **`proxy_read_timeout`** et **`proxy_send_timeout`** élevés (ex. 7200s).
4. **Certbot** `--nginx` sur le sous-domaine.
5. DNS chez le registrar : **A** du sous-domaine → IP du VPS.

Exemple de commandes prod Gunicorn (à adapter aux chemins) :

```text
gunicorn -w 1 --threads 4 -b 127.0.0.1:5000 --timeout 7200 app:app
```

---

## 5. YouTube / yt-dlp — pièges documentés

- **« Sign in to confirm you’re not a bot »** : cookies exportés + souvent Node + PO Token (bgutil) sur datacenter.
- **`n challenge` / formats manquants** : sans Node visible pour le worker Python → presque que des thumbnails ; corriger avec Node + `yt-dlp-ejs` et cookies.
- **Plusieurs workers Gunicorn** + état des jobs **en mémoire** (`job_states`) : SSE `/progress` et `/clip` sur des workers différents → jobs « perdus ». **Une seule instance worker** (ou Redis/etc. pas utilisé ici).
- **Long clips vs courts** : logique hybride dans le code (court → ffmpeg direct sur URLs ; long → yt-dlp subprocess + sections ; téléchargement **full** sans `player_client=android` pour garder **meilleure qualité**, Android limite les formats).
- **Annulation** : yt-dlp lancé en subprocess avec **nouveau groupe de processus** (`setsid`) + `killpg` pour tuer yt-dlp **et** les ffmpeg enfants.
- **Recommandation VPS** : `sudo apt install aria2 nodejs` (Node depuis Nodesource si besoin) ; Docker pour bgutil.

Maintenance détaillée : voir **`maintenance.md`** dans ce repo (cookies, mises à jour pip, conteneur bgutil).

---

## 6. Fichiers / dossiers importants dans le repo

| Chemin | Usage |
|--------|--------|
| `app.py` | Routes, jobs threads, yt-dlp, ffmpeg |
| `templates/` | `index.html`, `library.html` |
| `downloads/` | Fichiers temporaires clips / merges / sous-titres |
| `server_library/` | Backups MP4 + JSON si secret défini |
| `requirements.txt` | Dont `faster-whisper` |

---

## 7. API / flux principaux (référence rapide)

- `POST /clip` → `GET /progress/<job_id>` (SSE) → `GET /download/<job_id>`
- `POST /full-download` → `GET /full-download/progress/<id>` → `GET /full-download/file/<id>`
- `POST /cancel/<job_id>` — phases clip compatibles + PGID yt-dlp
- `POST /auth` + session cookie pour auto-save bibliothèque
- `POST /subtitles` + SSE progress + download
- Routes `/backup*` avec header `X-Backup-Key`

---

## 8. Pour ajouter **un autre service** sur le même VPS

- Nouvelle app sur un **autre port** localhost (ex. `127.0.0.1:8001`).
- Nouveau **server block Nginx** + nouveau sous-domaine + **Certbot** `-d autre.domaine`.
- Nouveau **unit systemd**. Pas besaire d’exposer le port de l’app sur Internet si tout passe par Nginx (80/443 seulement).

---

## 9. Ce document ne contient **aucun secret**

Remplacer IP, domaines et mots de passe par les tiens dans ta copie si tu édites ce fichier pour un autre projet.

---

*Généré pour faciliter la continuité de contexte entre le projet Clip_youtube et d’autres dépôts / chats.*
