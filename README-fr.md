[<img width="200" alt="get in touch with Consensys Diligence" src="https://user-images.githubusercontent.com/2865694/56826101-91dcf380-685b-11e9-937c-af49c2510aa0.png">](https://diligence.consensys.net)<br/>
<sup>
[[  🌐  ](https://diligence.consensys.net)  [  📩  ](mailto:diligence@consensys.net)  [  🔥  ](https://consensys.net/diligence/tools/)]
</sup><br/><br/>

# Smart Contract - Meilleurs pratique sécurité

Visiter le site de la documentation: https://consensys.github.io/smart-contract-best-practices/

Lire la documentation en chinois : https://github.com/ConsenSys/smart-contract-best-practices/blob/master/README-zh.md
Lire la documentation en vietnamien : https://github.com/ConsenSys/smart-contract-best-practices/blob/master/README-vi.md

## Les contributions sont les bienvenues!

Soumettez une pull request, que ce soit pour une petite résolution de bugs ou bien ou toutes nouvelles sections. Si vous écrivez du nouveau contenue, veuillez vous rendre sur la page de [contribution](./docs/about/contributing.md) pour de l'aide concernant le format.

Voir les [issues](https://github.com/ConsenSys/smart-contract-best-practices/issues) concernant des sujets qui doivent être traités ou bien mis à jour. Si vous avez une idée que vous aimeriez partgée, veuillez nous contacter sur [Gitter](https://gitter.im/ConsenSys/smart-contract-best-practices).

Si vous avez écrit un article ou bien un poste sur un blog, veuillez l'ajouter à la [bibliographie](./docs/bibliography.md).

## Charger le site de la documentation

```
git clone git@github.com:ConsenSys/smart-contract-best-practices.git
cd smart-contract-best-practices
pip install -r requirements.txt
mkdocs build 
```

Pour lancer le serveur (et relancer en cas d'échec):

```
until mkdocs serve; do :; done
```

Vous pouvez aussi utiliser la commande `mkdocs serve` pour voir le site en local sur votre poste, et voir tous vos changements en direct.

## Redeployer le site de la documentation

```
mkdocs gh-deploy
```
