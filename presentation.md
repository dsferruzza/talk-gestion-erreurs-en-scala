% Gérer les erreurs avec l'aide du système de types de Scala !
% David Sferruzza

# À propos de moi

- [\@d_sferruzza](https://twitter.com/d\_sferruzza)
- [github.com/dsferruzza](https://github.com/dsferruzza)
- développeur et responsable R&D chez [Escale](http://www.escaledigitale.com)
- doctorant en génie logiciel à l'Université de Nantes
- écrit des projets perso et pro en Scala et en Haskell (notamment) depuis ~ 2 an

![](img/escale.png)


# Cas d'exemple

On veut valider un utilisateur qui s'inscrit.

```scala
case class User(id: UUID,
                bio: String,
                birthday: String)

case class CheckedUser(id: UUID,
                       bio: String,
                       birthday: DateTime)

def validate(user: User): CheckedUser
```


# Cas d'exemple

```scala
case class User(id: UUID,
                bio: String,
                birthday: String)
```

3 échecs possibles :

- la bio est trop longue
- la date de naissance n'est pas au bon format
- la date de naissance est dans le futur


# Principe des exceptions

- on lance des exceptions depuis la définition de la fonction
- on espère qu'elles seront attrapées au niveau de l'appel
- YOLO

![](img/throw.gif)


# Problème avec les exceptions

- si on n'attrape pas l'exception, elle se propage
- si on oublie de l'attraper, ça compile quand même
- la signature de la fonction est trompeuse

```scala
def validate(user: User): CheckedUser
```

![](img/liar.gif)


# Option[A]

```scala
def validate(user: User): Option[CheckedUser] = {
  // Si tout est ok
  Some(checkedUser)

  // Sinon
  None
}
```

Suffisant si on n'a pas besoin d'information sur l'erreur


# Option[A] => A

```scala
val checkedUser: Option[CheckedUser] = ???

// Pas bien (erreur de compilation)
checkedUser.id

// Pas bien (peut lancer une exception)
checkedUser.get.id

// Bien
checkedUser match {
  case Some(u) => u.id
  case None    => ???
  // Obligé de traiter le cas où ça échoue
}
```


# ADT pour les erreurs

```scala
// Validation Error
sealed trait VE

case class BioTooLong(length: Int) extends VE
case object InvalidBirthdayFormat extends VE
case object ImpossibleBirthday extends VE
```

**sealed** aide le compilateur à vérifier tous les cas possibles


# Either[A, B]

```scala
def validate(user: User):
            Either[VE, CheckedUser] = {
  // Si tout est ok
  Right(checkedUser)

  // Sinon
  Left(error)
}
```


# Either[A, B]

```scala
val checkedUser: Either[VE, CheckedUser] = ???

checkedUser match {
  case Right(u)                    => u.id
  case Left(BioTooLong(length))    => ???
  case Left(InvalidBirthdayFormat) => ???
  case Left(ImpossibleBirthday)    => ???
  // Warning du compilateur si on oublie un cas
}
```


# Try[T]

```scala
def validate(user: User): Try[CheckedUser] = {
  // Si tout est ok
  Success(checkedUser)

  // Sinon
  Failure(error)
}
```

Similaire à Either :

- `Try[T] ~= Either[Throwable, T]`
- pas de liste exhaustive des erreurs (`Throwable`)
- sympa pour interagir avec du code qui peut lancer des exceptions (Java)


# Future[T]

Similaire à Try, mais asynchrone

```scala
val f: Future[T] = ???
val value: Option[Try[T]] = f.value
```

- `None` si pas encore complété
- `Some(Success(t))` si complété avec succès
- `Some(Failure(error))` si complété avec une erreur


# scalaz.\\/[A, B]

```scala
import scalaz.{\/, -\/, \/-}
def validate(user: User): VE \/ CheckedUser = {
  // Si tout est ok
  \/-(checkedUser)

  // Sinon
  -\/(error)
}
```

Similaire à Either, mais part du principe que la valeur intéressante est à droite (*right-biased*)

```scala
eitherVal.right.map(???)
disjunctionVal.map(???)
```


# scalaz.Validation[E, A]

```scala
import scalaz.{ValidationNel, Success, Failure}
import scalaz.syntax.validation._
import scalaz.syntax.applicative._
```

```scala
def checkBioLength(u: User):
                  ValidationNel[VE, User] = {
  val l = u.bio.length
  if (l < 5) u.success
  else BioTooLong(l).failureNel
}

def checkBirthdayFormat(u: User):
                ValidationNel[VE, User] = ???

```


# scalaz.Validation[E, A]

```scala
def validate(user: User):
            ValidationNel[VE, CheckedUser] = {
  val cbl = checkBioLength(user)
  val cbf = checkBirthdayFormat(user)
  (cbl |@| cbf) { (u, _) =>
    CheckedUser(???)
  }
}
```

Permet d'accumuler les erreurs lorsqu'on fait des validations indépendantes


# rapture.modes (principe)

- définir une fonction pouvant échouer et la wrapper avec Rapture
- au niveau de l'appel de la fonction, choisir le mode désiré (`import modes.returnEither._`)
- la fonction va renvoyer un `Either` !

GitHub : [propensive/rapture-core](https://github.com/propensive/rapture-core)

[http://blog.scalac.io/2015/05/28/scala-modes.html](http://blog.scalac.io/2015/05/28/scala-modes.html)


# Conclusion

Utiliser correctement ces types pour gérer les erreurs permet :

- d'afficher clairement le contrat d'une fonction (pure)
- d'avoir une vérification de cohérence par le compilateur

![](img/awesome.gif)


# Questions ?

![](img/question.gif)

Twitter : \@d_sferruzza

Slides sur GitHub :

[dsferruzza/talk-gestion-erreurs-en-scala](http://github.com/dsferruzza/talk-gestion-erreurs-en-scala)
