# Process d'installation pour utilisation d'un LLM local Ollama

La doc de référence est un billet de blog: https://www.linuxtricks.fr/wiki/ia-une-web-interface-a-ollama-avec-ollama-llm-ui-projet-en-dev

J'ai toutefois complété ces indications avec des infos spécifiques à notre config locale. Cette installation est dédiée à un système Debian/Ubuntu.

## Pré-requis nodejs/npm/nvm

Installer nodejs (et npm qui vient avec) ainsi que nvm pour la gestion de version de npm. On ajoute la version 20 de npm.

```bash
apt install nodejs
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
nvm version
nvm install 20
nvm alias default 20
```

## Installation de Ollama LLM UI

```bash
cd /opt
git clone https://github.com/Peanuts-83/Ollama-llm-ui
cd Ollama-llm-ui
# le .env contient l'IP du serveur qui fourni l'api (la petite tour du bureau)
# OLLAMA_URL="http://192.168.0.151:11400"
# nginx gère le reverse-proxy vers localhost:11434
npm install
npm run dev
```

A ce stade, on peut tester l'UI sur http://localhost:3000

## Setup du build pour tourner en prod

Dans le dépot, lancer le build pour générer e dossier **.next** optimisé.

```bash
npm run build
```

## Création d'un service systemd

Pour automatiser le service et qu'il soit lancé au démarrage du poste.

```bash
sudo nano /etc/systemd/system/ollama-llm-ui.service
```

Cette config diffère de celle fourni par la doc de référence, car il s'agit d'une config de prod.
Ajouter ces informations en ajustant à la config local du poste, en particulier les paths.

```bash
[Unit]
Description=Next.js Ollama LLM UI
After=ollama.service
Requires=ollama.service

[Service]
WorkingDirectory=/opt/Ollama-llm-ui
ExecStart=/home/tom/.nvm/versions/node/v20.19.0/bin/npm start
Restart=always
RestartSec=10
User=tom
Environment=PATH=/home/tom/.nvm/versions/node/v20.19.0/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

```bash
# Pour localiser npm
which npm
```

Relancer le daemon systemctl et le service.

```bash
systemctl daemon-reload
systemctl enable ollama-llm-ui.service
systemctl start ollama-llm-ui.service
systemctl status ollama-llm-ui.service
```

### Bonus: changer de port

Si on veut que le service écoute un autre port, modifier le fichier de config du service (exemple ici sur le port 8080).

```bash
ExecStart=/home/tom/.nvm/versions/node/v20.19.0/bin/npm start -- -p 8080
```

## Open WebUI

ref https://github.com/open-webui/open-webui <br>
ref https://docs.openwebui.com/

> Une alternative plus performante est fournie sur le dépot en ref, qui peut s'installer via une ligne de commande docker (libre à vous de créer un service pour automatiser le process si cette solution vous plait).

Les options utilisées ici sont les suivantes:
* -d detached
* -p expose le port 3001 (le port 300 est pris par Ollama-llm-ui)
* gpus activer calculs GPU si Nvidia dispo (si erreur driver [voir ce fix](https://stackoverflow.com/questions/75118992/docker-error-response-from-daemon-could-not-select-device-driver-with-capab))
* -e variable d'environnement qui pointe sur mon API locale
* -v database de backup des conversations
* --name le nom du conteneur
* --restart toujours
* ghcr.io/open-webui/open-webui:main nom de l'image source docker

```bash
docker run -d -p 3001:8080 --gpus all -e OLLAMA_BASE_URL=http://192.168.0.151:11400 -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

De nombreuses options sont possibles, cf la doc en ligne.
