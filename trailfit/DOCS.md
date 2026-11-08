# TrailFit

TrailFit et sa base PostgreSQL tournent dans cet add-on. Les données sont dans le dossier de l'add-on et sont
incluses dans les sauvegardes Home Assistant (l'add-on s'arrête une minute pendant la sauvegarde).

## Installation (une fois)

1. **Accès à l'image** : l'image est privée sur GitHub. Crée un jeton GitHub *classic* avec le seul droit
   `read:packages`, puis ajoute-le dans Paramètres › Modules complémentaires (« Applications » selon la version)
   › Boutique › ⋮ › Registres : registre `ghcr.io`, utilisateur `Joezzef`, mot de passe : le jeton.
2. **Dépôt** : dans Boutique › ⋮ › Dépôts, ajoute exactement `https://github.com/Joezzef/trailfit-addon`
   (sans `.git` ni `/` final : le nom d'hôte interne de l'add-on en dépend).
3. **Reprise des données du PC (facultatif)** : avant le **premier** démarrage, dépose `restore.sql` (créé sur le
   PC par `pnpm addon:dump`) dans le partage `share`, sous `trailfit/restore.sql`, par exemple avec l'add-on
   Samba share. Il n'est chargé qu'au premier démarrage (l'add-on ne peut que lire ce partage) : supprime-le
   ensuite, il contient toutes tes données. Si ce premier démarrage est interrompu, il recommence de zéro au
   suivant.
4. Installe **TrailFit**, remplis la configuration, démarre-le et active **Mise à jour automatique** et
   **Chien de garde** (relance si l'application ne répond plus).

L'add-on ne démarre pas tant que l'adresse publique n'est pas remplie.

## Configuration

| Option | Rôle |
| --- | --- |
| Adresse publique | Obligatoire. L'adresse HTTPS du tunnel Cloudflare, par exemple `https://trailfit.mondomaine.fr`. Les liens des e-mails s'en servent. |
| Serveur, port, utilisateur et mot de passe SMTP | Envoi des e-mails (confirmation d'adresse, mot de passe oublié). Gmail : `smtp.gmail.com`, port `587` (chiffrement exigé), ton adresse et un [mot de passe d'application](https://myaccount.google.com/apppasswords). Facultatif : sans SMTP, l'admin crée les comptes dans Profil › Mon compte › Gérer les comptes (mot de passe provisoire à transmettre) et y redonne un mot de passe en cas d'oubli. |
| Expéditeur | Par exemple `TrailFit <ton.adresse@gmail.com>`. |
| Administrateur | Ce compte devient administrateur, le seul à utiliser la clé Garmin personnelle. S'il n'existe pas, il est créé et reçoit un e-mail pour choisir son mot de passe (si l'envoi échoue, un mot de passe provisoire s'affiche une fois dans le journal). |
| Clé de chiffrement | Facultatif : `APP_ENCRYPTION_KEY` du `.env` du PC, pour que la clé Garmin reprise reste lisible. Sinon, recolle ta clé dans Profil › Garmin. |
| Clé API Claude | Facultatif : active le coach IA. |
| Exploitant, Contact | Facultatif : affichés sur la page « Tes données » (qui gère ce TrailFit, comment le joindre). |

## Tunnel Cloudflare

TrailFit n'ouvre aucun port sur le réseau de la maison : le tunnel l'atteint par le réseau interne de Home
Assistant, à l'adresse `http://9aeb3281-trailfit:3000` (nom d'hôte également affiché sur la page de l'add-on).

- **Tunnel géré dans l'add-on Cloudflared** : ajoute dans l'option `additional_hosts` :

  ```yaml
  additional_hosts:
    - hostname: trailfit.mondomaine.fr
      service: http://9aeb3281-trailfit:3000
  ```

- **Tunnel géré depuis le tableau de bord Cloudflare** (jeton Zero Trust) : `additional_hosts` est ignoré. Crée
  le nom d'hôte public dans Zero Trust › Networks › Tunnels, avec le service `http://9aeb3281-trailfit:3000`.

Tant que tu es seul à utiliser TrailFit, tu peux protéger l'adresse avec Cloudflare Access.

Le port 3000 peut être activé dans l'onglet Réseau pour voir le site depuis le réseau local, mais la connexion
n'y fonctionne pas (adresse différente de l'adresse publique, cookies réservés au HTTPS) : passe par le tunnel.

## Mises à jour

Chaque push sur `main` du dépôt de TrailFit publie une nouvelle version.

- **Avec le webhook (conseillé)** : la mise à jour s'installe une minute après la fin du déploiement.
- **Sans webhook** : Home Assistant voit la nouvelle version dans les 3 heures, mais n'installe une mise à jour
  automatique qu'au moins 24 heures après l'avoir vue (contrôle toutes les 16 heures). Une version plus récente
  repousse ce délai : si tu pousses tous les jours, rien ne s'installe tout seul.

Home Assistant fait une sauvegarde de l'add-on avant chaque mise à jour. Si une version pose problème, restaure
cette sauvegarde : elle remet l'ancienne version et ses données. Ne supprime donc jamais d'anciennes versions de
l'image sur GitHub.

### Webhook

Crée cette automatisation. Remplace l'identifiant du webhook par une longue chaîne aléatoire, et vérifie le nom
de l'entité de mise à jour de TrailFit dans Paramètres › Entités :

```yaml
alias: TrailFit — installer la nouvelle version
triggers:
  - trigger: webhook
    webhook_id: trailfit-deploy-REMPLACER-PAR-UNE-CHAINE-ALEATOIRE
    allowed_methods: [POST]
    local_only: false
actions:
  # demande à Home Assistant de relire le dépôt, puis attend que la version annoncée apparaisse
  - action: homeassistant.update_entity
    target:
      entity_id: update.trailfit_update
  - wait_template: "{{ state_attr('update.trailfit_update', 'latest_version') == trigger.json.version }}"
    timeout: "00:05:00"
    continue_on_timeout: false
  - action: update.install
    target:
      entity_id: update.trailfit_update
    data:
      backup: true
```

Puis, dans le dépôt GitHub `trailfit`, ajoute le secret `HA_WEBHOOK_URL` :
`https://<adresse publique de Home Assistant>/api/webhook/<identifiant du webhook>`. Si Home Assistant est
protégé par Cloudflare Access, ce chemin doit rester accessible.

## Sécurité

Home Assistant installe les mises à jour sans redemander d'accord, même si une version réclamait plus de droits.
Ce qui tourne sur le Pi dépend donc de ton compte GitHub : active la double authentification et protège la
branche `main` du dépôt `trailfit`.
