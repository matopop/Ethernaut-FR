Pour gagner ce niveau, il faut :
- Obtenir la propriété du contrat
- Réduire sa balance à 0

Ce qui pourrait aider :
- Trouver comment envoyer des ether en interagissant avec l'ABI,
- Trouver comment envoyer des ether en dehors de l'ABI,
- Convertir les wei en ether et inversement (voir help())
- Comprendre ce que sont les fallback functions


De ce qu'on voit dans le code fournit, c'est qu'il y a plusieurs fonctions :

- Les fonctions avec un modifier onlyOwner :

  - withdraw() qui est une fonction publique mais qui a un modifier onlyOwner qui va vérifier que le msg.sender soit l'owner. Du coup on ne pourra pas s'en servir tant qu'on sera pas owner.

- Les fonctions sans modifier et accessible librement :
  - getContribution() qui permet de voir les contributions du msg.sender, quand on l'appelle on voit qu'on a contribué 0
  - contribute() qui va vérifier que la valeur du message soit inférieure à 0.001 ether. Cette fonction va ajouter au mapping contributions la contribution que lui fait le msg.sender. Si la contribution du msg.sender est plus grande que la contribution du owner, alors l'owner devient le msg.sender

C'est dans contribute() qu'il y a notre faille :)

On voit, dans le constructor, que la contribution du msg.sender à l'origine est de 1000 ether, il faudra donc trouver un moyen de faire comprendre au contrat qu'on lui envoie plus de 1000 ether.

On peut vérifier cela en faisant ça :

```
await contract.contributions(await contract.owner())
```

Pour l'avoir en plus lisible en string :

```
(await contract.contributions(await contract.owner())).toString();

-> 1000000000000000000000
```

A savoir que là ca nous donne le résultat en wei.

Si on transforme en eth, en faisant :

`montanteth = montantwei / 10^18`

Du coup, on voit qu'il a 1000 ether en contributions.

Ca risque d'être compliqué d'envoyer 1000 ether pour avoir la propriété du contrat.

Ce qui serait plus simple est de vérifier sur le reste du code.

## Les fonctions fallback

Lorsque l'on s'intéresse à comprendre ce que sont les fallback functions en solidity, on tombe là dessus : https://www.geeksforgeeks.org/solidity-fall-back-function/

En l'occurence, cela nous explique qu'une fallback fonction est une fonction executée seulement si les autres fonctions ne match pas avec ce qui est appelé/envoyé au contrat.

Par exemple, si on utilise une fonction qui n'existe pas avec ce contrat, ça va simplement aller exécuter la fonction fallback.

Une fonction fallback est une fonction :
- `external`
- Sans nom
- Sans argument
- Qui ne retourne rien
- Qui ne peut être défini qu'une seule fois dans un contrat
- Qui est executée si l'appelant appelle une fonction non écrite dans le contrat
- Limitée à 2300 gas quand elle est appelée par une autre fonction, pour qu'elle soit le plus cheap possible

Pour recevoir des ether, la fonction doit être écrite avec le keyword `receive`. De ce fait, elle est automatiquement vue comme `payable`.
Cela donne ça :

```
receive() external payable{
// ce qu'elle fait
}
```

En l'occurence ici, cette fonction indique :

```
receive() external payable {
  require(msg.value > 0 && contributions[msg.sender] > 0);
  owner = msg.sender;
}
```

Donc, si on fait quelque chose qui n'est pas indiqué dans le code, comme envoyer une transaction ou appeler une fonction qui n'existe pas tout en envoyant une transaction (la fonction fallback ici nous indique qu'il faut que ce soit une transaction ayant une valeur supérieure à 0 et que le mapping contributions soit supérieur à 0), alors ca rend le msg.sender désormais owner, toujours selon cette fallback function.

Donc en faisant simplement :
- Une contribution avec la fonction contribute() tel quel:

```
contract.contribute.sendTransaction({value: toWei(0.0009});
```
cela va mettre dans contributions(notreAdresse) qu'on a contribué 0.0009

Puis faire ça pour devenir owner:

```
contract.sendTransaction({value: 1});
```

Puis on peut voir que désormais, on est owner:

```
await contract.owner()

// -> Va imprimer notre adresse
```

Du coup, désormais, on peut appeler toutes les fonctions onlyOwner, là il y a seulement withdraw, du coup on va pouvoir appeler la fonction withdraw et tout s'envoyer :). C'est cette fonction :

```
function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }
```
Cette fonction va transférer à l'owner la balance complète.

Donc plus qu'a l'appeler :
```
contract.withdraw()
```

et hop ca va tout nous envoyer :)
