# https-github.com-MHZDeveloper-Oracle-blob-master-tp2-ReadMe.md
# TP2

Vous trouverez dans ce [lien](https://docs.google.com/presentation/d/1f5uyqowZ7u9QAV5YUseURFAPnu4v6_O7cnL8T1eZTIo/edit?usp=sharing) la présentation utilisée dans ce TP.

### Script de démarrage

```
DELETE FROM EMP WHERE ENAME IN ('Hichem','Mohamed');

Insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values ('7839','Mohamed','PLEASE',null,to_date('17/11/81','DD/MM/RR'),'2000',null,'10');
Insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values ('7698','Hichem','CRAFTSMAN','7839',to_date('01/05/81','DD/MM/RR'),'2800',null,'10');
```

## Introduction

Après avoir présenté les transactions de façon générale, nous allons présenter dans ce chapitre la gestion de la conccurence entre les transactions.

<p align="center">
  <img width="650" src="images/1.gif" alt="picture">
</p>

Tout d'abord nous présenterons les anomalies causées par cette concurrence puis nous présenterons les différents types d'isolation pour y remédier.

## Concurrence : Anomalies des Transactions

Il y a 3 types d'anomalies : 

### 1/ Mises à jour perdues

Elle se produit lorsque **deux transactions lisent une même donnée et la modifient, l’une après l’autre => une des écritures est perdues** .

Prenons par exemple 2 utilisateurs (transactions) qui veulent réserver un nombre de place pour un spectacle : 

<p align="center">
  <img width="600" src="images/2.png" alt="picture">
</p>


Lorsque ces 2 transactions lisent en même temps le nombre de place disponible les 2 vont lire **10 places**.
Ensuite chacune va réserver un nombre de place.
Une va réserver **2 places** et l'autre **4 places** .
Après que ces transactions soient exécutées, le nombre de place restant est de **6** hors que ca devrait être **4 places**.

### 2/ Lectures non répétables

Elle se produit lorsqu' **une transaction lit une même donnée 2 fois et nous constatons que la deuxième lecture est différente de la première**. 

Nous continuons avec le même exemple précédent :  

<p align="center">
  <img width="600" src="images/3.png" alt="picture">
</p>


Lorsqu'une transaction consulte le nombre de place disponible et qu'entre temps une autre transaction réserve **2 places**.
Tandis qu'après un traitement, la première transaction consulte à nouveau le nombre de place celui-ci s'avère modifié.


### 3/ Lectures sales

Elle se produit lorsqu' **il y a un entrelacement des transactions qui empêchent la bonne exécutions des Commit/Rollback**. 

Nous continuons avec le même exemple :  

<p align="center">
  <img width="600" src="images/4.png" alt="picture">
</p>


Lorsque 2 transactions consultent le nombre de place disponible et que l'une d'entre elle réserve et fait un **Commit** tandis que l'autre après un traitement décide de réserver des places puis fait un **Rollback**.
Le dernier **Rollback** ne s'exécutera pas correctement.

### 4/ Interblocages

Biensûr, lorsqu'on parle de gestion de conccurence entre plusieurs transactions, il est possible qu'un interblocage se produise.

<p align="center">
  <img width="600" src="images/5.png" alt="picture">
</p>

### Demo Interblocages :

| Timing | Session N° 1 (User1)   | Session N° 2 (User2) |Résultat | 
| :----: | :----: |:----:|:----:|
| t0 | ``` SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem');``` ||On obtient les noms et les salaires des employés 'Mohamed' et 'Hichem'<br>.Mohamed:2000 et Hichem:2800|
| t1 | ``` UPDATE EMP SET SAL = 4000 WHERE ENAME ='Hichem'; ``` |------|On va réaliser dans la session N°1 , une modification du salaire de l'employé 'Hichem' qui va passer de 2800 à 4000. Un lock sera ajouté par Oracle sur la ligne après avoir exécuté la requête UPDATE|
| t2 | ------ |```UPDATE EMP SET SAL = SAL + 1000 WHERE ENAME ='Mohamed';```|On va réaliser une modification dans la session N°2 sur le salaire de l'employé 'Mohamed'. On va ajouter 1000 à son salaire (Son salaire sera égal à 3000).Après la réalisation de cette requête UPDATE, un lock sera ajouté à la ligne modifiée.|
| t3 | ```UPDATE EMP SET SAL = SAL + 1000 WHERE ENAME ='Mohamed';```| ------ |Puisqu'il existe un lock sur la ligne concernant l'employé 'Mohamed'(réalisé à partir de la session N°2 à l'aide de la requête UPDATE),la session N°1 va rester en attente jusqu'à ce que q'un COMMIT soit exécuté dans la session N°2 afin que le lock soit levé et ainsi une modification pourra être réalisée par le User1 relatif à la session N°1 sur les données correspondantes à l'employé 'Mohamed'|
| t4 | ------ |```UPDATE EMP SET SAL = SAL + 1000 WHERE ENAME ='Hichem';```|La session 1 va detecter l'interblocage |
| t5 | ```Commit;``` |------| Session 2: --> 1 row updated.|
| t6  |```UPDATE EMP SET SAL = SAL + 1000 WHERE ENAME ='Mohamed';```| ------|Le lock sur la ligne concernant l'employé l'employé 'Mohamed' n'a pas encore été levé car il y a eu une requête UPDATE (t2) qui n'a pas été suivie d'un COMMIT donc le lock est toujours présent (créé à partir de la session N°2) donc la session N°1 va rester en attente.|
| t7 |  ------ |```Commit;```|En réalisant un COMMIT dans la session N°2, le verrou sur la ligne concernant l'employé 'Mohamed' sera libéré et ainsi on aura le message 1 row updated dans la session N°1 indiquant la bonne réalisation de la requête UPDATE|
| t8 | ------ |```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem', 'Maaoui');```|On aura comme résultat de la requête, une table contenant les noms 'Hichem' et 'Mohamed' suivis respectivement par les salaires 3000 et 5000 ( le resultat obtenu devrait être respectivement 4000 et 5000) car il y a eu réalisation de la requête UPDATE EMP SET SAL=SAL+1000 where ENAME='Mohamed' deux fois une fois dans la session N°1 et une fois dans la session N°2 (2000+1000+1000=4000). Afin de corriger cette incohérence , il faut taper un COMMIT dans la session N°1 afin de lever le lock de la ligne concernant l'employé 'Mohamed' et ainsi obtenir un résultat correct.|

## Concurrence : Niveaux d'isolation des transactions

Plus le niveau est permissif, plus l’exécution est fluide, plus les anomalies sont possibles.
Plus le niveau est strict, plus l’exécution risque de rencontrer des blocages, moins les anomalies sont possibles.
Le niveau d’isolation par défaut n’est jamais le plus strict c'est pourquoi quand l’isolation totale est nécessaire, il faut l’indiquer explicitement.

Les 4 niveaux existants dans Oracle sont  **(ordonnés du moins strict au plus strict)** : 

- **Read uncommited** : tout est permis 

- **Read commited** : une requête accède à l’état de la base de données *au moment où la requête est exécutée*

- **Repeatable read** : Une requête accède à l’état de la base de données *au moment où la transaction a débutée*

- **Serializable** : Garantit une isolation totale => Cohérence de la base.

Parfois, le niveau le plus strict rejette des transactions voire provoquer des interblocages.
C'est pourquoi Oracle munit les développeurs de la clause ***FOR UPDATE***.
Autrement dit, le développeur déclare qu’une lecture va être suivie d’une mise à jour et le système pose un verrou fort pour celle-là, pas de verrou pour les autres et ainsi ca permet de monter le niveau de blocage uniquement quand c’est nécessaire... mais ca repose sur le facteur humain qui n'est pas assez fiable.

### Demo Niveau d'isolation  READ COMMITTED 

| Timing | Session N° 1 | Session N° 2 | Résultat | 
| :----: | :----: |:----:|:----:|
| t0 | ``` SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','');``` ||On obtient le salaire de l'employé 'Mohamed'.<br> Mohamed:2000|
| t1 | ``` UPDATE EMP SET SAL = 4000 WHERE ENAME ='Hichem'; ``` |------|On va faire passer le salaire de l'employé 'Hichem' à 4000 à l'aide d'une requête UPDATE dans la session N°1(un lock sera ajouté sur cette ligne par Oracle)|
| t2 | ------ |```SET TRANSACTION ISOLATION LEVEL READ COMMITTED;```|On va régler le niveau de la transaction sur READ COMMITTED dans la session N°2.**NB**:Cette instruction doit être réalisée pour chaque transaction.|
| t3 | ------ |```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem');```|On va obtenir le nom et le salaire des employés 'Mohamed' et 'Hichem' à partir de la table EMP dans la session N°2.Le salaire de l'employé 'Hichem' ne passe pas à 4000 lors de l'affichage (à partir de la requête UPDATE (t1) réalisé dans la session N°1) car il n'y a pas eu de COMMIT dans la session N°1 et donc il existe un verrou sur la ligne de l'employé 'Hichem'.<br> Mohamed:2000 Hichem:2800|
| t4 | ------ |```UPDATE EMP SET SAL = 3800 WHERE ENAME ='Mohamed';```|A partir de la session N°2, le salaire de l'employé 'Mohamed va passer à 3800, avec cela un lock sera ajouté à la ligne mise à jour.|
| t5 | ```Insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,COMM,DEPTNO) values ('9999','Maaoui','Magician',null,to_date('17/02/2021','DD/MM/RR'),null,'10');``` |------|A partir de la session N°1,on va insérer dans la table EMP un nouvel employé ayant pour EMPNO(9999), pour ENAME('Maaoui'), pour JOB('Magician'), pour MGR(null) pour HIREDATE(17/02/2021), pour COMM(null) et pour DEPTNO(10)|
| t6 | ------ |```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem', 'Maaoui');```|On va afficher les noms et les salaires des employés 'Mohamed','Hichem' et 'Maaoui' à partir de la session N°2. L'employé 'Maaoui' a été inséré dans la session N°1 sans la réalisation d'un COMMIT après son insertion donc il ne sera pas affiché dans la session N°2.<br>Mohamed:3800 Hichem:2800 |
| t7 | ------ |```UPDATE EMP SET SAL = 5000 WHERE ENAME ='Hichem';```|Puisqu'il existe un lock sur la ligne concernant l'employé 'Hichem' car une requête UPDATE a été réalisée dans la session N°1(t1) donc la session N°2 va rester en attente.|
| t8 | ```Commit;``` |------|En réalisant le COMMIT dans la session N°1, la transaction va être validée et le lock sur la ligne concernant l'employé 'Hichem' sera levé et donc la requête UPDATE réalisée dans la session N°2 va être effectuée(le message 1 row updated sera affiché).|
| t9 | ------ |```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem', 'Maaoui');```|On va afficher à l'aide de la requête le nom et les salaires des employés 'Hichem' 'Mohamed' et 'Maaoui' dans la session N°1. L'employé 'Maaoui' sera affiché cette fois car il y a eu validation de la transaction dans la session N°1.<br>Mohamed:3800 Hichem    :5000,Maaoui:|
| t10 | ------ |```COMMIT;```|On va réaliser un COMMIT ainsi, les verrous sur les lignes concernant les employés 'Hichem' et 'Mohamed' seront levés.|
| t11 | ```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem', 'Maaoui');```|------|On va afficher encore une fois les salaires et les noms des employés 'Hichem','Mohamed' et 'Maaoui' dans la session N°1(On va obtenir un résultat correct car le COMMIT réalisé dans t10 a permis de lever les verrous sur les différentes lignes sur lesquelles on a effectué une mise à jour et donc les modifications seront conservées).|




### Demo Niveau d'isolation SERIALIZABLE ;

| Timing | Session N° 1  | Session N° 2 |Résultat | 
| :----: | :----: |:----:|:----:|
| t0| ``` SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem');``` |||
| t1| ``` UPDATE EMP SET SAL = 4000 WHERE ENAME ='Hichem'; ``` |------|On va modifier le salaire de l'employé 'Hichem' à 4000 dans la session N°1 à l'aide d'une requête UPDATE.|
| t2| ------ |```SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;```|On va régler le niveau de la transaction sur Serializable dans la session N°2.|
| t3| ------ |```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem');```|On va afficher le nom et le salaire des employés 'Hichem' et 'Mohamed' dans la session N°2.|
| t4| ------ |```UPDATE EMP SET SAL = 3800 WHERE ENAME ='Mohamed';```|En exécutant la requête UPDATE dans la session N°2, une copie de la base de données sera créée dans le cache d'Oracle.La requête UPDATE sera réalisée sur cette copie(Le salaire de l'employé Hichem passe à 3800).|
| t5| ```Insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,COMM,DEPTNO) values ('9999','Maaoui','Magician',null,to_date('17/02/2021','DD/MM/RR'),null,'10');``` |------|On va réaliser l'insertion d'une ligne dans la table EMP à partir de la session N°1|
| t6| ```COMMIT;```|------ |On valide la transaction dans la session N°1 à l'aide de l'instruction COMMIT(Les modifications réalisées sur la base de données sont sauvegardées).|
| t7|```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem', 'Maaoui');```| ------ |On va afficher les noms et les salaires des employés 'Hichem', 'Mohamed' et 'Maaoui' dans la session N°1.|
| t8| ------ |```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem', 'Maaoui');```|On va afficher maintenant les noms et les salaires des employés 'Hichem', 'Mohamed' et 'Maaoui' (dans la session N°2) qui seront similaires à ceux affichés dans la session N°1 car l'ordre COMMIT dans t6 a validé la transaction.|
| t9| ```Commit;``` |------|On va valider la transaction à l'aide d'un COMMIT dans la session N°1|
| t10|```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem', 'Maaoui');```| ------ |On va afficher dans la session N°1 les noms et les salaires des employés 'Hichem', 'Mohamed' et 'Maaoui'(Il n'y aura pas de modification lors de l'affichage par rapport au dernier ordre select).|
| t11| ------ |```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem', 'Maaoui');```|A l'aide d'un select dans la session N°2, on va effectuer la sélection des noms et des salaires tel que les noms des employés sont 'Hichem' , 'Mohamed' et 'Maaoui'.|
| t12| ------ | ```COMMIT;```|On va valider la transaction (serializable) dans la session N°2.|
| t13| ``` UPDATE EMP SET SAL = 5000 WHERE ENAME ='Maaoui'; ``` |------|On va modifier le salaire de l'employé 'Maaoui' à 5000 dans la session N°1. Après l'éxecution de la requête update relative au changement du salaire de l'employé 'Maaoui', un lock va être ajouté à sa ligne|
| t14| ------ |```SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;```|On va régler la  transaction sur le niveau serializable dans la session N°2|
| t15| ------ |```UPDATE EMP SET SAL = 5200 WHERE ENAME ='Maaoui';```|On va tenter de modifier le salaire de l'employé 'Maaoui' à 5200 dans la session N°2. La session N°2 va détecter l'interblocage ( le niveau de la transaction est serializable et la base a été copiée dans le cache avec le lock car dans t13, on a réalisé une requête sur l'employé 'Maaoui' dans il y aura un lock dans la base sur la ligne le concernant).|
| t16| ```COMMIT;``` |------|On valide la transaction dans la session N°1 à l'aide d'un commit afin de lever le lock.|
| t17| ------ |```ROLLBACK;```|On effectue un rollback dans la session N°2 afin d'annuler les modifications réalisées( la base copiée dans le cache Oracle contient un lock sur la ligne de l'employé 'Maaoui', donc on ne pourra pas modifier les données de cet employé car en réalisant un commit pour valider la transaction dans la session N°1, le lock sur la copie de la base persistera mais , il sera levé dans la base réelle.|
| t18| ------ |```SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;```|On va régler encore une fois le niveau de la transaction sur serializable dans la session N°2.|
| t19| ------ |```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem', 'Maaoui');```|A l'aide d'une requête select, on affiche les noms et les salaires des employés 'Hichem' , 'Mohamed' et 'Maaoui'.|
| t20| ``` UPDATE EMP SET SAL = 5200 WHERE ENAME ='Maaoui'; ``` |------|Le salaire de l'employé 'Maaoui' est mis à jour (Son salaire passe à 5200) à l'aide de la requête UPDATE réalisée dans la session N°1.|
| t21| ```COMMIT;``` |------|On va réaliser un COMMIT afin de valider la transaction et lever les locks sur la ligne de l'employé 'Maaoui'. Ainsi quand il y aura copie de la base grâce au niveau de la transaction Serializable , elle ne contiendra aucun lock et les requêtes UPDATE dans la session N°2 seront effectués avec succès sur l'ensemble de la base.|
| t22| ------ | ```COMMIT;```|On exécute un COMMIT dans la session N°2 afin de valider la transaction ayant le niveau Serializable et ainsi les données stockées dans la copie de la base(dans le cache Oracle) vont être chargées dans la base réelle|
| t23| ------ |```SELECT ENAME, SAL FROM EMP WHERE ENAME IN ('Mohamed','Hichem', 'Maaoui');```|On va afficher dans la session N°2 les noms et les nouvelles valeurs des salaires des employés 'Mohamed','Hichem' et 'Maaoui' après la validation de la transaction dans t21 et t22.|



### Exemple de traitement avec "FOR UPDATE" ;

```sh
SET SERVEROUTPUT ON ;

DECLARE 

tmp_v_empno number (6,2);
CURSOR cur IS SELECT EMPNO FROM EMP WHERE ENAME = 'Mohamed' FOR UPDATE of SAL;

BEGIN 
   DBMS_OUTPUT.PUT_LINE('START');

   OPEN cur;   

    FETCH cur INTO tmp_v_empno;

      IF cur%notfound THEN
          tmp_v_empno := -1 ; 
          DBMS_OUTPUT.PUT_LINE('This Employee is currently being Updated, try again in a while');
      ELSE
          DBMS_OUTPUT.PUT_LINE(' Updating Employee Number  ' || tmp_v_empno );
          UPDATE EMP SET SAL = 6666 WHERE CURRENT OF cur ;
          COMMIT;
    END IF;

   CLOSE cur;

   DBMS_OUTPUT.PUT_LINE(' Employee Number  ' || tmp_v_empno || ' Updated successfully !!'  );

END;
/

```

Pour vérifier :
```sh
SELECT EMPNO, SAL FROM EMP WHERE ENAME = 'Mohamed' ;
```
## Conclusion

- Comprendre et utiliser correctement les niveaux d’isolation est impératif pour les applications transactionnelles

- Savoir repérer les transactions dans une application. Elle doivent respecter lacohérence applicative(le système ne peut pas la deviner pour vous)

- La clausefor updateest en théorie meilleure, mais repose sur le facteur humain

- Savoir qu’en mode sérialisable on risque des rejets

- Comprendre les risques dans les autres modes.
