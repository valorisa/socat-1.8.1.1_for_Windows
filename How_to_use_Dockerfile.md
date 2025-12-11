# Procédure

## 1. Sauvegarder le Dockerfile
Dans le dossier de ton app (où se trouve `requirements.txt` et le module `socat/example.py`) :

```bash
ls
# tu dois voir : Dockerfile  requirements.txt  socat/  ...
```

Assure‑toi que le contenu du `Dockerfile` est bien celui généré.

## 2. Build de l’image
Depuis ce dossier :

```bash
docker build -t socat-api:1.8.1.0 .
```

- L’étape `wget + ./configure + make + make install` pour socat prendra un peu de temps la première fois.
- Si ça casse, colle la fin des logs de build.

## 3. Lancer le container
```bash
docker run --rm -p 8181:8181 --name socat-api socat-api:1.8.1.0
```

- Le container doit afficher les logs de gunicorn (workers en écoute sur 0.0.0.0:8181).[1][2]

## 4. Tester depuis l’hôte
Dans un autre terminal :

```bash
curl http://127.0.0.1:8181/
```

Tu dois recevoir la réponse de `socat.example:app`.  
Si tu veux, envoie le contenu de `requirements.txt` et de `socat/example.py` pour vérifier que la cible `socat.example:app` est correcte.

[1](https://github.com/benoitc/gunicorn/issues/2138)
[2](https://stackoverflow.com/questions/58429866/running-gunicorn-as-non-root-user)
