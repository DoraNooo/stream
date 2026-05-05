# Stream ultrawide — OBS → VPS (`stream.barium-saas.fr`)

Objectif : envoyer **un seul flux** depuis ton PC (OBS) vers ton VPS, regarder dans le navigateur sur **HTTPS**, sans dépendre des presets 16:9 des plateformes grand public. Les **plusieurs résolutions** (vraie adaptive pour des écrans différents) viennent ensuite avec du **transcodage** (CPU/GPU sur le VPS ou second flux OBS).

---

## Pourquoi pas Twitch (résumé)

- Contrôle total sur **canvas 32:9** et débits ; pas de cadre imposé par la plateforme.
- Pas de modération / discoverability imposée ; bout du tunnel chez toi.
- Tu paies ton VPS ; pas de partenaire ni contraintes uploads tiers.

---

## Architecture (MVP actuel)

```text
[OBS chez toi] --RTMP (1935/tcp)--> [VPS : nginx-rtmp Docker]
                                         |
                                         +--> segments HLS (/hls/...)
                                                  ^
[Nginx host HTTPS] ------------------------------+
    |
    +--> fichiers statiques / lecteur (player/)
```

- **Ingest** : `rtmp://stream.barium-saas.fr:1935/live/<clé_secrète>`
- **Lecture** : `https://stream.barium-saas.fr/` + saisie de la clé (ou `?key=...` dans l’URL pour tes essais ; pour des amis, mieux vaut une **page sans afficher la clé** ou une auth basique plus tard).

---

## Déploiement sur le VPS

1. **DNS** : enregistrement **A** `stream.barium-saas.fr` → IP du VPS.
2. **Pare-feu** : ouvrir **443** (public), **1935/tcp** (public ou restreint à ton IP fixe si tu en as une).
3. Copier ce dépôt (ou au minimum `docker-compose.yml`, `deploy/nginx-stream.conf`, `player/`).
4. Lancer le conteneur :

   ```bash
   cd /chemin/vers/stream
   docker compose up -d
   ```

5. Déployer le lecteur sur le disque servi par Nginx (exemple) :

   ```bash
   sudo mkdir -p /var/www/stream-player
   sudo cp -r player/* /var/www/stream-player/
   ```

6. **Nginx host** : fusionner `deploy/nginx-stream-host.example.conf` avec ta config (chemins SSL réels pour Certbot). Puis :

   ```bash
   sudo nginx -t && sudo systemctl reload nginx
   ```

---

## OBS — réglages conseillés (32:9)

- **Canvas / sortie** : alignés sur ton ratio natif (ex. **3840×1080** ou **2560×720**).
- **Encodeur** : x264 ou NVENC/AMD selon ton GPU ; **format RTMP** inchangé.
- **Bitrate** : à la hausse avec la résolution (ex. **12–20 Mbps** pour du 1440p ultrawide jeu vidéo si ta ligne montante le permet ; sinon réduis résolution ou FPS).
- **URL du serveur** : `rtmp://stream.barium-saas.fr:1935/live`
- **Clé de diffusion** : une longue chaîne aléatoire (comme un mot de passe).

Si la lecture coupe ou buffer : baisse bitrate ou fragment HLS (fichier `deploy/nginx-stream.conf`, directives `hls_fragment` / `hls_playlist_length`).

---

## Multi-résolutions plus tard

| Approche | Idée |
|----------|------|
| **Transcodage VPS** | `exec ffmpeg` dans nginx-rtmp (comme l’image par défaut `alfg/nginx-rtmp`, mais avec des `-vf scale` / crops adaptés au **32:9**, pas aux presets 720p 16:9). Coûteux en CPU. |
| **Deux sorties OBS** | Plugin multi-sorties : deux RTMP vers deux applications (`live_hd`, `live_sd`) si ta montée débit le permet. |
| **Un second flux** | Deux services Docker ou deux `application` RTMP, deux pages ou sélecteur côté joueur. |

Le MVP actuel est **une seule qualité** : tous les spectateurs reçoivent le même flux ; sur un écran 16:9 ils verront des bandes ou du zoom selon le réglage du lecteur / CSS.

---

## Rapport avec l’autre service (Clip_youtube)

Même VPS : autre **hostname**, autre **ports localhost**, autre **unit systemd** ou **compose**. Pas besoin d’exposer le port **8088** : il reste sur `127.0.0.1`, seul le Nginx du host parle au conteneur.

---

## Sécurité rapide

- Clé RTMP **longue et secrète** ; rotation si fuite.
- Idéal : **restreindre 1935** à ton IP ; sinon surveillance des logs Nginx / connexions.
- Ne partage pas publiquement une URL avec `?key=` en production ; prévois auth ou jetons courts ensuite.
